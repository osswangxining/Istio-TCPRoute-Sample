As we’ll see throughout the rest of this book, Istio will allow us to solve some difficult challenges in service-to-service communication. For most of the book, we’ll assume a single cluster with a single Istio control-plane deployment, but in reality Istio’s capabilities are not limited to a single or homogeneous cluster. But even before we look at multi-cluster or hybrid deployments, we should understand how to connect different networks together. This chapter will consider two different networks: the cluster in which the service mesh is deployed and where user services are deployed, and anything outside of the cluster.

We will most likely run interesting services and applications inside our cluster. We will most likely have intra-service communication within the cluster and that’s where Istio shines. But what about those clients that are deployed or exist outside of the cluster? In this chapter, we’ll take a look at connecting those clients that live outside the cluster to services running inside the cluster.

4.1  Traffic ingress concepts
The networking community has a term for connecting networks via well established entry points called ingress points. Ingress refers to traffic that originates outside the local network and is intended for an endpoint within the local network. This traffic is first routed to exposed ingress points for the purpose of enforcing rules and policies about what traffic is allowed into the local network. If traffic does not go through these ingress points, it cannot connect with any of the services inside the cluster. If the ingress point allows the traffic, it proxies it to the correct endpoint in the local network. If the traffic is not allowed, the ingress point refuses to proxy the traffic.

4.1.1  Virtual IPs: simplifying service access
At this point, it’s useful to dig in a little bit more to how traffic is routed to a network’s ingress points, at least how it relates to the type of clusters at which we’ll be looking in this book. Let’s say we have an API exposed at api.istioinaction.io/v1/products that our customers can use to get a list of products in our catalog. When our client tries to query that endpoint, the client’s networking stack first tries to resolve the api.istioinaction.io domain name to an IP address. This is done with DNS servers. Their networking stack would query the DNS servers for what the IP address should be, which means we need to map our service’s IP to this name when we register it in DNS. For a public address, we could use a service like Amazon Route 53 or Google Cloud DNS and map a domain name to an IP address. In our own datacenters, we’d use our internal DNS servers to do the same thing. But to what IP address should we map the name?

We are probably not going to map the name directly to a single instance or single endpoint of our service (single IP) as that can be very fragile. What would happen if that one specific service instance goes down? Clients would see a lot of errors until we changed the DNS mapping to a new IP address with a working endpoint. But doing this any time a service goes down is slow, error prone, and low availability.

What we want to do is map the domain name to an IP address that represents our service. This single "virtual IP" would represent our service to be able to forward traffic to our actual service instances. We will map the domain name to a virtual IP that’s bound to a type of ingress point known as a reverse proxy. The reverse proxy is an intermediary component that’s responsible for distributing requests to backend services and does not correspond to any specific service. The reverse proxy can also provide capabilities like load balancing so requests don’t overwhelm any one specific backend.

4.1.2  Virtual Hosting: multiple services from a single access point
Foo: In the previous section we saw how a single virtual IP can be used to address a service that may be comprised of many actual service instances with their own IP, however, the client only uses the virtual IP. We can also represent multiple different services using a single virtual IP. For example, we could have both prod.istioinaction.io and api.istioinaction.io resolve to the same virtual IP address. This means requests for both URIs would end up going to the same virtual IP and thus the same ingress reverse proxy. If the reverse proxy was smart enough, it could use the Host HTTP header to further delineate which requests should go to which group of services.

Hosting multiple different services at a single entry point is known as virtual hosting. We need some way to decide to which virtual-host group a particular request should be routed. With HTTP/1.1, we can use the Host header, with HTTP/2 we can use the :authority header, and with TCP connections we can rely on Server Name Indication (SNI) with TLS. We’ll take a closer look at SNI later in this chapter. The important thing to note is that the edge ingress functionality we’ll see in Istio uses both virtual IP routing as well as virtual hosting to route service traffic into the cluster.

4.2  Istio Gateway
Istio has a concept of an ingress Gateway that plays the role of the network-ingress point and is responsible for guarding and controlling access to the cluster from traffic that originates outside of the cluster. Additionally, Istio’s Gateway also plays the role of load balancing and virtual-host routing.


In Figure XX we see that by default Istio uses an Envoy proxy as the ingress proxy. We saw in Chapter 3 that Envoy is a capable service-to-service proxy, but it can also be used to load balance and route proxy traffic from outside the service mesh to services running inside of it. All of the features of Envoy that we saw in the previous chapter are also available in the ingress gateway.

Let’s take a closer look at how Istio uses Envoy to implement an ingress gateway. When we installed Istio in Chapter 2, we saw the listing of components that make up the control plane and any additional components that support the control plane.

If we do a listing of the Kubernetes pods in the istio-system namespace (where the control plane was installed), we should see the istio-ingressgateway component:

You can see in the "READY" column for the ingress-gateway-xxx component that it has 1/1 containers ready while some of the other ones have 2/2. Some of the other components have a service proxy injected alongside them similar to how our services will have (as discussed in Chapter 2). This allows those components in the control plane to take advantage of the service-mesh capabilities. The istio-ingressgateway component, however, has only a single Envoy proxy container deployed so we see 1/1.

NOTE
In the above listing, right next to the istio-ingressgateway pod, you may notice the istio-egressgateway component. This component is responsible for routing traffic out of the cluster. We’ll cover that in more depth in Chapter XX on traffic management.

If you’d like to verify that the Istio service proxy is indeed running in the Istio ingress gateway, you can run something like this:

$  INGRESS_POD=$(kubectl get pod -n istio-system \
| grep ingressgateway | cut -d ' ' -f 1)

$  kubectl -n istio-system exec $INGRESS_POD  ps aux


We should see a process listing as the output showing the Istio service proxy command line with both the discovery-agent and the envoy processes.

At this point, although we’ve got a running Envoy playing the role of the Istio Gateway, we have no configuration or rules about what traffic we should let into the cluster. We can verify that we don’t have anything by running the following commands:

$  istioctl -n istio-system proxy-config listener $INGRESS_POD
Error: no listeners found

$  istioctl -n istio-system proxy-config route $INGRESS_POD
Error: config dump has no route dump


To configure Istio’s Gateway to allow traffic into the cluster and through the service mesh, we’ll start by exploring two concepts: Gateway and VirtualService. Both are fundamental in general to getting traffic to flow in Istio, but we’ll look at them only within the context of allowing traffic into the cluster. We will cover VirtualService more fully in the next chapter about traffic management and routing.


4.2.1  Specifying Gateway resources
To configure a Gateway in Istio, we use the Gateway resource and specify which ports we wish to open on the Gateway, and what virtual hosts to allow for those ports. The example Gateway we’ll explore is quite simple and exposes an HTTP port on port 80 that will accept traffic destined for virtual host apiserver.istioinaction.io

apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: coolstore-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "apiserver.istioinaction.io"


This Gateway definition is intended for the istio-ingressgateway that was created when we set up Istio initially, but we could have used our own definition of Gateway. We can define to which gateway the configuration applies by using the labels in the selector section of the Gateway configuration. In this case, we’re selecting the gateway implementation with the label istio: ingressgateway which will match the default istio-ingressgateway. The Gateway for istio-ingressgateway is really just an instance of Istio’s service proxy (Envoy), and its Kubernetes deployment configuration looks something like this:

containers:
- name: ingressgateway
  image: "gcr.io/istio-release/proxyv2:1.0.0"
  imagePullPolicy: IfNotPresent
  ports:
    - containerPort: 80
    - containerPort: 443
  args:
  - proxy
  - router
  - -v
  - "2"
  - --serviceCluster
  - custom-ingressgateway
  - --proxyAdminPort
  - "15000"
  - --discoveryAddress
  - istio-pilot.istio-system:8080


Our Gateway resource configures Envoy to listen on port 80 and expect HTTP traffic. Let’s create that resource and see what it does. From the root of the source code that accompanies this book, you should see a chapter-files/chapter4/coolstore-gw.yaml file and you can create the resource like this:

$  kubectl create -f chapter-files/chapter4/coolstore-gw.yaml

Let’s see whether our settings took affect.

$  istioctl proxy-config listener $INGRESS_POD  -n istio-system

ADDRESS     PORT     TYPE
0.0.0.0     80       HTTP

$  istioctl proxy-config route $INGRESS_POD -o json  -n istio-system

[
    {
        "name": "http.80",
        "virtualHosts": [
            {
                "name": "blackhole:80",
                "domains": [
                    "*"
                ],
                "routes": [
                    {
                        "match": {
                            "prefix": "/"
                        },
                        "directResponse": {
                            "status": 404
                        },
                        "perFilterConfig": {
                            "mixer": {}
                        }
                    }
                ]
            }
        ],
        "validateClusters": false
    }
]

We see our listener is bound to a "blackhole" default route that just routes everything to HTTP 404. In the next section, we’ll take a look at setting up a virtual host for routing traffic from port 80 to a service within the service mesh.

Before we go to the next section there’s an important last point to be made here. The pod running the gateway, whether that’s the default istio-ingressgateway or your own custom gateway, will need to be able to listen on a port or IP that is exposed outside the cluster. For example, on our local minikube that we’re using for these examples, the ingress gateway is listening on a NodePort A NodePort uses a real port on one of the Kubernetes clusters' nodes. On minikube there is only one node and its the VM that minikube runs on. If you’re deploying on a cloud service like GKE, you’ll want to make sure you use a LoadBalancer that gets an externally routable IP address. More information can be found at https://istio.io/docs/tasks/traffic-management/ingress/.

4.2.2  Gateway routing with Virtual Services
So far, all we’ve done is configured the Istio Gateway to expose a specific port, expect a specific protocol on that port, and define specific hosts to serve from the port/protocol pair. When traffic comes into the gateway, we need a way to get it to a specific service within the service mesh and to do that, we’ll use the VirtualService resource. In Istio, a VirtualService resource defines how a client talks to a specific service through its fully qualified domain name, which versions of a service are available, and other routing properties (like retries and request timeouts). We’ll cover VirtualService in more depth in the next chapter when we explore traffic routing more deeply, but in this chapter it’s sufficient to know that VirtualService allows us to route traffic from the ingress gateway to a specific service.

An example of a VirtualService that routes traffic for the virtual host apigateway.istioinaction.io to services deployed in our service mesh looks like this:


With this VirtualService resource, we define what to do with traffic when it comes into the gateway. In this case, as you can see with spec.gateways field, these traffic rules apply only to traffic coming from the coolstore-gateway gateway definition which we created in the previous section. Additionally, we’re specifying a virtual host of apiserver.istioinaction.io for which traffic must be destined for these rules to match. An example of matching this rule is a client querying apiserver.istioinaction.io which resolves to an IP that the Istio gateway is listening on. Additionally, a client could explicitly set the Host header in the HTTP request to be apiserver.istioinaction.io which we’ll show through an example.

First, let’s create this VirtualService and explore how Istio exposes this on the gateway:

After a few moments (it may take a few for the configuration to sync; recall in previous sections that configuration in the Istio service mesh is eventually consistent), we can re-run our commands to list the listeners and routes :

The output for the route should look similar to the previous listing, although it may contain other attributes and information. The critical part is we can see how defining a VirtualService created an Envoy route in our Istio Gateway that routes traffic matching domain apiserver.istioinaction.io to service apiserver in our service mesh.

This configuration assumes you have the sample application deployed from Chapter 2, but if you do not, run this to install the apigateway and catalog services with Istio’s service proxy injected alongside the services

Once all the pods are ready, you should see something like this:


Verify that your Gateway and VirtualService are installed correctly:

Now, let’s try to call the gateway and verify the traffic is allowed into the cluster:


We should actually see no response. Why is that? If we take a closer look at the call by printing the headers, we should see that the Host header we sent in is not a host that the gateway recognizes.

The Istio Gateway, nor any of the routing rules we declared in VirtualService knows anything about Host: 192.168.64.27:31380 but it does know about virtual host apiserver.istioinaction.io. Let’s override the Host header on our command line and then the call should work:

Now you should see a successful response.

4.2.3  Overall view of traffic flow
In the previous subsections, we got hands on with the Gateway and VirtualService resources from Istio. The Gateway resource defines our ports, protocols, and virtual hosts that we wish to listen for at the edge of our service mesh cluster. The VirtualService resources define where traffic should go once it’s allowed in at the edge. In Figure 4.7 we see the full end-to-end flow:

4.2.4  Istio Gateway vs Kubernetes Ingress
When running on Kubernetes, you may ask "why doesn’t Istio just use the Kubernetes Ingress resource to specify ingress?". In some of Istio’s early releases there was support for using Kubernetes Ingress, but there are significant drawbacks with the Kubernetes Ingress specification.

The first issue is that the Kubernetes Ingress is a very simple specification geared toward HTTP workloads. There are implementations of Kubernetes Ingress (like NGINX, Heptio Contour, etc) however each of them is geared toward HTTP traffic. In fact, Ingress specification really only considers port 80 and port 443 as ingress points. This severely limits the types of traffic a cluster operator can allow into the service mesh. For example, if you have Kafka or NATS.io workloads, you may wish to expose direct TCP connections to these message brokers. Kubernetes Ingress doesn’t allow for that.

Second, the Kubernetes Ingress resource is severely underspecified. There is no common way to specify complex traffic routing rules, traffic splitting, or things like traffic shadowing. The lack of specification in this area causes each vendor to re-imagine how best to implement configurations for each type of Ingress implementation (HAProxy, Nginx, etc).

Lastly, since things are underspecified, the way most vendors chose to expose configuration is through bespoke annotations on deployments. The annotations between vendors varied and were not portable, and if Istio continued that trend there would have been many more annotations to account for all the power of Envoy as an edge gateway.


Ultimately, Istio decided on a clean slate for building ingress patterns and specifically separating out the layer 4 (transport) and layer 5 (session) properties from the layer 7 (application) routing concerns. Istio Gateway handles the L4 and L5 concerns while VirtualService handles the L7 concerns.

4.3  Securing Gateway traffic
So far, we’ve shown how to expose basic HTTP services with the Istio gateway using the Gateway and VirtualService resources. When connecting services from outside of a cluster (let’s say, the public internet) to those running inside a cluster, one of the basic capabilities of the ingress gateway in a system is to secure traffic and help to establish trust in the system. One way we can begin to secure our traffic is to give clients confidence that the service they’re hoping to communicate with is indeed the service it claims to be. Additionally, we want to exclude anyone from eavesdropping on our communication, so we should encrypt the traffic.


Istio’s gateway implementation allows us to terminate incoming TLS/SSL traffic, pass it through to the backend services, redirect any non-TLS traffic to the proper TLS ports as well as implement mutual TLS. We’ll take a look at each of these capabilities in this section.

4.3.1  HTTP traffic with TLS
To prevent man-in-the-middle (MITM) attacks and to encrypt all traffic coming into the service mesh, we can set up TLS on the Istio gateway so that any incoming traffic is served over HTTPS (for HTTP traffic; we’ll cover non-HTTP traffic in the next sections). MITM attacks occur when a client connects to a service, but instead of the service it intends, it connects to a impostor service claiming to be the intended service. The impostor service can gain access to the communication including sensitive information. TLS helps to mitigate this attack.

To enable TLS/HTTPS for our ingress traffic, we need to specify the correct private keys and certificates that the gateway should use. As a quick reminder, the certificate that the server presents is how it announces its identity to any clients. The certificate is basically the server’s public key that has been signed by a reputable authority also known as a Certificate Authority (CA). For a client to trust that the server’s certificate is indeed valid, it must have first installed the CA issuer’s certificate. This way a client can verify that the certificate is signed by a CA that it trusts. The private key that the server uses is how it encrypts traffic it sends to the client who can then decrypt the traffic with the server’s public key (which can be found in the certificate).


Before we can configure the default istio-ingressgateway to use certificates and keys, we need to make them available to the service proxy running inside of istio-ingressgateway. If we were to take a close look at the default deployment for istio-ingressgateway, we’ll see that it’s already mounting in Kubernetes volumes that should host the certificates and keys:

We see that the Kubernetes secret named istio-ingressgateway-certs is mounted in path /etc/istio/ingressgateway-certs on the istio-ingressgateway pod. By default, this istio-ingressgateway-certs secret does not exist, so in order to set up HTTPS/TLS, we will need to create this secret.

Luckily in the source files that accompany this book, we have example certificates and keys to use for this exercise. A huge shout out to Nicholas Jackson from HashiCorp for creating the tools that we used to generate these certificates (github.com/nicholasjackson/mtls-go-example). For those interested in generating the certificates and keys yourself, please take a look at the openssl command line tool.

Let’s start by creating the istio-ingressgateway-certs secret:

Now we can configure out Istio Gateway resource to use the certificates and keys:



As we can see in our Gateway resource in listing XX, we’ve opened port 443 on our ingress gateway, and we’ve specified its protocol to be HTTPS. Additionally, we’ve added a tls section to our gateway configuration where we’ve specified the locations to find the certificates and keys to use for TLS. Note, these are the same locations that were mounted into the istio-ingressgateway that we saw earlier.

Let’s replace our gateway with this new Gateway resource:


Now if we try to connect to our service on port 443, we should end up with an error (note, when running on minikube and the ingress gateway is exposed on a NodePort, it won’t be port 443 but rather some other port that gets mapped to port 443 on the istio-ingressgateway container).

First thing we should figure out the correct URL for the minikube NodePort on which the istio-ingressgateway is listening. On another installation of Kubernetes you may be using either NodePort or LoadBalancer. For cloud LoadBalancer implementations, you should figure out the IP address that gets assigned to your LoadBalancer and use port 443 on that IP. On minikube, we can do the following to figure out the right HTTPS URL:

If we call the service like we did in the previous section, by passing in the proper Host header, we should see something like this (note, we use https:// in the URL):


This means the Certificate presented by the server cannot be verified using the default CA certificate chains. Let’s pass in the proper CA certificate chain to our curl client:



The client still cannot verify the certificate! This is because the server certificate is issued for apiserver.istioinaction.io and we’re calling the minikube IP (192.168.64.27 in this case). We can use a curl parameter called --resolve that lets us call the service as though it was at apiserver.istioinaction.io but then tell curl to use the minikube IP and port:


Now we should see a proper HTTP/1.1 200 response and the JSON payload for the products list. As a client we’re verifying the server is who it says it is by trusting the CA that signed the certificate and we’re able to encrypt the traffic to the server by using this certificate.


NOTE
Note, for curl to work in this section, you need to make sure it supports TLS and you can add your own CA certificates to override the default. Not all builds of curl support TLS. For example, in some versions of curl on Mac OS X, CA certificates can only come from the Apple Keychain. Newer builds of curl should have the proper SSL libraries and should work for you, but what you want to see is something about your SSL library (OpenSSL, LibreSSL, etc) when you type this:

curl --version | grep -i SSL


At this point, we’ve secured traffic by encrypting it to the Istio ingress gateway, which terminates the TLS connection and then sends the traffic to the backend apigateway service running in our service mesh. The hop between the istio-ingressgateway component and the apigateway service is not encrypted or secured in anyway yet. We will cover securing internal service-mesh traffic in Chapter XX.

4.3.2  HTTP redirect to HTTPS
We set up TLS in the previous section, but what if we want to force all traffic to always use TLS? In the previous section we could have used both http:// and https:// to access our service through the ingress gateway, but in this section we want to force all traffic to use HTTPS. To do that, we have to modify our Gateway resource slightly to force a redirect for HTTP traffic.


If we update our Gateway to use the above configuration, we can limit all traffic to only HTTPS.



Let’s set up the correct http:// URL to use:

Now if we call the ingress gateway on the HTTP port, we should see something like this:

Now we can expect all traffic going to our ingress gateway to always be encrypted.

4.3.3  HTTP traffic with mutual TLS
In the previous section, we used standard TLS to allow the server to prove its identity to the client, but what if we want our cluster to verify who the clients are before we accept any traffic from outside the cluster? In the simple TLS scenario, the server sends its public certificate to the client and the client verifies that it trusts the CA that signed the server’s certificate. What we want to do is have the client send its public certificate and let the server verify that it trusts it. When the client and server each verify the other’s certificates and use these to encrypt traffic, we call this mutual TLS (mTLS).

To configure the default istio-ingressgateway to participate in a mutual TLS connection, we need to give it a set of Certificate Authority (CA) certificates to use to verify a client’s certificate. Just like we did in the previous section, we need to make this CA certificate (or certificate chain more specifically) available to the istio-ingressgateway with a Kubernetes secret.

Let’s start by configuring the istio-ingressgateway-ca-certs secret with the proper CA certificate chain:

Now let’s update the Istio Gateway resource to point to the location of the CA certificate chain as well as configure the expected protocol to be mutual TLS:


Let’s replace the Gateway configuration with this new updated version:

Now if we try to call the ingress gateway the same way we did in the previous section (i.e., assuming simple TLS), we should see the call rejected:

We see this call getting rejected because the SSL handshake wasn’t successful. We are only passing the CA certificate chain to the curl command, we need to also pass the client’s certificate and private key. With curl we can do it by passing the --cert and --key parameters like this:

Now we should see a proper HTTP/1.1 200 response and the JSON payload for the products list. The client is both validating the server’s certificate as well as sending its own certificate for validation to achieve mutual TLS.


4.3.4  Serving multiple virtual hosts with TLS
Istio’s ingress gateway can serve multiple virtual hosts each with its own certificate and private key from the same HTTPS port (i.e., port 443). To do that, we just add multiple entries for the same port and the same protocol. For example, we can add multiple entries for both the apiserver.istioinaction.io and catalog.istioinaction.io services each with their own certificate and key pair. An Istio Gateway resource that describes multiple virtual hosts served with HTTPS looks like this:

Notice that both of the entries each listen on port 443, each serve the HTTPS protocol, but they have different names. One is named https-apiserver, and the other is named https-catalog. Each has its own unique certificates and keys that are used for the specific virtual host it serves. To put this into action, we need to add these new certificates and keys. Let’s create them:

Now we want to update the istio-ingressgateway to mount this new certificate/key pair from the /etc/istio/catalog-ingressgateway-certs/ folder. We use an istio-ingressgateway deployment that’s already been updated to this:

Now that we have the new secrets and the gateway is mounting them, we can update the gateway configuration:

Lastly, we need to add a VirtualService for the catalog service that we’ll be exposing through this ingress gateway:


Now that we’ve updated the istio-ingressgateway, let’s give it a try. Calling apiserver.istioinaction.io should work just like it did in the simple TLS section:

Now when we call the catalog service through the Istio gateway, let’s use different certificates:

Both calls should succeed with the same response. You may be wondering how does the Istio ingress gateway know which certificate to present depending who’s calling? In other words, there’s only a single port opened for these connections, how does it know which service the client is trying to access and which certificate corresponds with that service? The answer lies in an extension to TLS called Server Name Indication (SNI). Basically, when a HTTPS connection is created, the client first identifies which service it’s trying to reach using the ClientHello part of the TLS handshake. Istio’s Gateway (Envoy specifically) implements SNI on TLS which is how it’s able to present the correct cert and route to the correct service. For more on SNI, see XXX

In this section we successfully exposed to different virtual hosts and served each with its own unique certificate through the same HTTPS port. In the next section, we’ll take a look at TCP traffic.

4.4  TCP traffic

Istio’s gateway is powerful enough to serve not only HTTP/HTTPS traffic, but any traffic accessible via TCP. For example, we can expose a database (like MongoDB) or a message queue like Kafka through the ingress gateway. When Istio treats the traffic as plain TCP, we do not get as many useful features like retries, request-level circuit breaking, complex routing, etc. This is simply because Istio cannot tell what protocol is being used (unless a specific protocol that Istio understands is used — like MongoDB). Let’s take a look at how to expose TCP traffic through the Istio Gateway so that clients on the outside of the cluster can communicate with those running inside the cluster.

4.4.1  Exposing TCP ports on the Istio Gateway
The first thing we need to do is create a TCP-based service within our service mesh. For this example, we’ll use the simple echo service from https://github.com/cjimti/go-echo/. This simple TCP service will allow us to login with a simple TCP client like telnet and issue commands that should be displayed back to us.

Let’s deploy the TCP service and inject the Istio service proxy next to it:

Next, we should create an Istio Gateway resource that exposes a specific non-HTTP port for this service. In the following example, we expose port 31400 on the default istio-ingressgateway. Just like with the HTTP ports, 80 and 443, this TCP port 31400 must be made available either as a NodePort or as a cloud LoadBalancer. In our examples running on minikube, this is exposed as a NodePort running on 31400:


Let’s create the gateway:

Now that we’ve exposed a port on our ingress gateway, we need to route the traffic to the echo service. To do that, we’ll use the VirtualService resource like we did in the previous sections. Note, for tcp traffic like that, we must match on the incoming port, in this case port 31400:

Let’s create the virtual service:

Now that we have exposed a port on our ingress gateway, and set up routing, we should be able to connect with a very simple telnet command:

As you type anything into the console and hit Return/Enter, you should see your text replayed back to you:

To quit from telnet press "ctl+]" and type quit and then hit Return/Enter.

4.4.2  Traffic routing with SNI an TLS
Gregor, this chapter is already getting long; should I do this section? I’m leaning toward yes… I’ve started to pull together an example just in case using a simple MQTT broker.

In this last section of the chapter, we learned how to use the Istio gateway functionality to accept and route non HTTP traffic, specifically, applications that may communicate over a TCP protocol that is very application specific. This opens the door for a much wider swath of applications that can participate in the service mesh including databases, message queues, caches, and even legacy applications with their own wire protocols. Istio’s gateway can be used to secure the traffic, route the traffic to backend services, and even passthrough secure traffic and allowing the backing services to handle the security handshakes it may use.

4.5  Summary
In this section we looked at why it’s important to have very fine-grained control over what traffic enters your service-mesh cluster and how to use the Istio Gateway and VirtualService constructs to do this. Specifically we saw these ways to restrict traffic:

By specific host over HTTP
By simple TLS/HTTPS to encrypt traffic and prevent man-in-the-middle attacks
Mutual TLS to provide identity to both server and client about who’s making the connections
Basic TCP routing for non-HTTP traffic
SNI/TLS for TCP connections for securing TCP connections
We started to explore VirtualService resource files to configure how routing happens at the ingress of our cluster. In the next chapter, we’ll expand on our understanding of VirtualService resources for the purposes of more powerful routing within the service mesh and how this control helps us control new deployments, route around failures, and implement powerful testing capabilities.
