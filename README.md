# Jersey S/MIME #

[![Build Status](https://secure.travis-ci.org/joschi/jersey-smime.png?branch=master)](https://travis-ci.org/joschi/jersey-smime)

Jersey S/MIME is a port of the S/MIME functionality in `resteasy-security` to the Jersey framework.

S/MIME (Secure/Multipurpose Internet Mail Extensions) is a standard for public key encryption and signing of MIME data. MIME data being a set of headers and a message body. It's most often seen in the email world when somebody wants to encrypt and/or sign an email message they are sending across the internet. It can also be used for HTTP requests as well which is what the Jersey integration with S/MIME is all about. Jersey S/MIME allows you to easily encrypt and/or sign an email message using the S/MIME standard.


## Maven Artifacts ##

You must include the jersey-crypto project to use the S/MIME framework.

    <dependency>
        <groupId>com.github.joschi</groupId>
        <artifactId>jersey-smime</artifactId>
        <version>0.2.0-SNAPSHOT</version>
    </dependency>


## Usage ##

### Message Body Encryption ###

While HTTPS is used to encrypt the entire HTTP message, S/MIME encryption is used solely for the message body of the HTTP request or response. This is very useful if you have a representation that may be forwarded by multiple parties and you want to protect the message from prying eyes as it travels across the network. Jersey S/MIME has two different interfaces for encrypting message bodies. One for output, one for input. If your client or server wants to send an HTTP request or response with an encrypted body, it uses the `com.github.joschi.jersey.security.smime.EnvelopedOutput` type. Encrypting a body also requires an X509 certificate which can be generated by the Java `keytool` command-line interface, or the `openssl` tool that comes installed on many OS's.

Here's an example of using the `EnvelopedOutput` interface:

    // server side   
    
    @Path("encrypted")
    @GET
    public EnvelopedOutput getEncrypted() {
       Customer cust = new Customer();
       cust.setName("Bill");
       
       X509Certificate certificate = ...;
       EnvelopedOutput output = new EnvelopedOutput(cust, MediaType.APPLICATION_XML_TYPE);
       output.setCertificate(certificate);
       return output;
    }


    // client side
    
    X509Certificate cert = ...; 
    Customer cust = new Customer();
    cust.setName("Bill");
    EnvelopedOutput output = new EnvelopedOutput(cust, "application/xml");
    output.setCertificate(cert);
    ClientResponse res = request.body("application/pkcs7-mime", output).post();

An `EnvelopedOutput` instance is created passing in the entity you want to marshal and the media type you want to marshal it into. So in this example, we're taking a Customer class and marshalling it into XML before we encrypt it. Jersey will then encrypt the `EnvelopedOutput` using the BouncyCastle framework's S/MIME integration. The output is a Base64 encoding and would look something like this:

    Content-Type: application/pkcs7-mime; smime-type=enveloped-data; name="smime.p7m"
    Content-Transfer-Encoding: base64
    Content-Disposition: attachment; filename="smime.p7m"
    
    MIAGCSqGSIb3DQEHA6CAMIACAQAxgewwgekCAQAwUjBFMQswCQYDVQQGEwJBVTETMBEGA1UECBMK
    U29tZS1TdGF0ZTEhMB8GA1UEChMYSW50ZXJuZXQgV2lkZ2l0cyBQdHkgTHRkAgkA7oW81OriflAw
    DQYJKoZIhvcNAQEBBQAEgYCfnqPK/O34DFl2p2zm+xZQ6R+94BqZHdtEWQN2evrcgtAng+f2ltIL
    xr/PiK+8bE8wDO5GuCg+k92uYp2rLKlZ5BxCGb8tRM4kYC9sHbH2dPaqzUBhMxjgWdMCX6Q7E130
    u9MdGcP74Ogwj8fNl3lD4sx/0k02/QwgaukeY7uNHzCABgkqhkiG9w0BBwEwFAYIKoZIhvcNAwcE
    CDRozFLsPnSgoIAEQHmqjSKAWlQbuGQL9w4nKw4l+44WgTjKf7mGWZvYY8tOCcdmhDxRSM1Ly682
    Imt+LTZf0LXzuFGTsCGOUo742N8AAAAAAAAAAAAA

Decrypting an S/MIME encrypted message requires using the `com.github.joschi.jersey.security.smime.EnvelopedInput` interface. You also need both the private key and `X509Certificate` used to encrypt the message. Here's an example:

    // server side
    
    @Path("encrypted")
    @POST
    public void postEncrypted(EnvelopedInput<Customer> input) {
        PrivateKey privateKey = ...;
        X509Certificate certificate = ...;
        Customer cust = input.getEntity(privateKey, certificate);
    }


    // client side
    
    ClientRequest request = new ClientRequest("http://localhost:9095/smime/encrypted");
    EnvelopedInput input = request.getTarget(EnvelopedInput.class);
    Customer cust = (Customer)input.getEntity(Customer.class, privateKey, cert);

Both examples simply call the `getEntity()` method passing in the `PrivateKey` and `X509Certificate` instances requires to decrypt the message. On the server side, a generic is used with `EnvelopedInput` to specify the type to marshal to. On the server side this information is passed as a parameter to `getEntity()`. The message is in MIME format: a `Content-Type` header and body, so the `EnvelopedInput` class now has everything it needs to know to both decrypt and unmarshall the entity.


### Message Body Signing ###

S/MIME also allows you to digitally sign a message. It uses the `multipart/signed` data format which is a multipart message that contains the entity and the digital signature.

Jersey S/MIME has two different interfaces for creating a `multipart/signed` message. One for input, one for output. If your client or server wants to send an HTTP request or response with an `multipart/signed` body, it uses the `com.github.joschi.jersey.security.smime.SignedOutput` type. This type requires both the `PrivateKey` and `X509Certificate` to create the signature. Here's an example of signing an entity and sending a `multipart/signed` entity.

    // server-side
     
    @Path("signed")
    @GET
    public SignedOutput getSigned() {
        Customer cust = new Customer();
        cust.setName("Bill");
    
        SignedOutput output = new SignedOutput(cust, MediaType.APPLICATION_XML_TYPE);
        output.setPrivateKey(privateKey);
        output.setCertificate(certificate);
        return output;
    }


    // client side
     
    ClientRequest request = new ClientRequest("http://localhost:9095/smime/signed");
    Customer cust = new Customer();
    cust.setName("Bill");
    SignedOutput output = new SignedOutput(cust, "application/xml");
    output.setPrivateKey(privateKey);
    output.setCertificate(cert);
    ClientResponse res = request.body("multipart/signed", output).post();


An `SignedOutput` instance is created passing in the entity you want to marshal and the media type you want to marshal it into. So in this example, we're taking a `Customer` class and marshalling it into XML before we sign it. Jersey will then sign the `SignedOutput` using the BouncyCastle framework's S/MIME integration. The output would look something like this:

    Content-Type: multipart/signed; protocol="application/pkcs7-signature"; micalg=sha1;  boundary="----=_Part_0_1083228271.1313024422098"
    
    ------=_Part_0_1083228271.1313024422098
    Content-Type: application/xml
    Content-Transfer-Encoding: 7bit
    
    <customer name="bill"/>
    ------=_Part_0_1083228271.1313024422098
    Content-Type: application/pkcs7-signature; name=smime.p7s; smime-type=signed-data
    Content-Transfer-Encoding: base64
    Content-Disposition: attachment; filename="smime.p7s"
    Content-Description: S/MIME Cryptographic Signature
    
    MIAGCSqGSIb3DQEHAqCAMIACAQExCzAJBgUrDgMCGgUAMIAGCSqGSIb3DQEHAQAAMYIBVzCCAVMC
    AQEwUjBFMQswCQYDVQQGEwJBVTETMBEGA1UECBMKU29tZS1TdGF0ZTEhMB8GA1UEChMYSW50ZXJu
    ZXQgV2lkZ2l0cyBQdHkgTHRkAgkA7oW81OriflAwCQYFKw4DAhoFAKBdMBgGCSqGSIb3DQEJAzEL
    BgkqhkiG9w0BBwEwHAYJKoZIhvcNAQkFMQ8XDTExMDgxMTAxMDAyMlowIwYJKoZIhvcNAQkEMRYE
    FH32BfR1l1vzDshtQvJrgvpGvjADMA0GCSqGSIb3DQEBAQUABIGAL3KVi3ul9cPRUMYcGgQmWtsZ
    0bLbAldO+okrt8mQ87SrUv2LGkIJbEhGHsOlsgSU80/YumP+Q4lYsVanVfoI8GgQH3Iztp+Rce2c
    y42f86ZypE7ueynI4HTPNHfr78EpyKGzWuZHW4yMo70LpXhk5RqfM9a/n4TEa9QuTU76atAAAAAA
    AAA=
    ------=_Part_0_1083228271.1313024422098--


To unmarshal and verify a signed message requires using the `com.github.joschi.jersey.security.smime.SignedInput` interface. You only need the `X509Certificate` to verify the message. Here's an example of unmarshalling and verifying a `multipart/signed` entity.

    // server side

    @Path("signed")
    @POST
    public void postSigned(SignedInput<Customer> input) throws Exception {
        Customer cust = input.getEntity();
        if (!input.verify(certificate)) {
           throw new WebApplicationException(500);
        }
    }


    // client side
    
    ClientRequest request = new ClientRequest("http://localhost:9095/smime/signed");
    SignedInput input = request.getTarget(SignedInput.class);
    Customer cust = (Customer)input.getEntity(Customer.class)
    input.verify(cert);


Contributors
------------

* Raffael Stein (@mrmaloke)


License
-------

This library is licensed under the Apache License, Version 2.0.

See http://www.apache.org/licenses/LICENSE-2.0.html or the LICENSE file in this repository for the full license text.
