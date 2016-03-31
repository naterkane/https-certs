--------------------------------------------------------------

HTTPS Authorized Certs with Node.js
===================================

If you build Node.js HTTPS servers as much as we do, you'll know how easy it is to get things going. But we were surprised to find that we could quickly add client x.509 certificate checking in just a few lines of code.

Typically, HTTPS servers do a basic TLS handshake and accept any client connection as long as a compatible cipher suite can be found. However, the server can be configured to challenge the client with a **CertificateRequest** during the TLS handshake. This forces the client to present a valid certificate before the negotiation can continue. This strategy can be used in API services instead of (or in addition to) another form of identity such as shared secrets or OAuth.

Here's some background on how [Client-authenticated TLS Handshakes](http://en.wikipedia.org/wiki/Transport_Layer_Security#Client-authenticated_TLS_handshake) work over at Wikipedia.

From Scratch
------------

Let's walk through the process of creating certificates and build an HTTPS server and client to use them. First we'll build a Certificate Authority to sign our own client certificates. (let's also use it to sign our server certificate so we don't have to pay a public certificate authority!)

If you're on a mac, there's a good chance that you may get a `unable to write 'random state'` error when running openssl... to fix this issue just create and env variable called `RANDFILE` and point it to `.rnd` in your working directory 

    export RANDFILE=$(pwd)/.rnd 

To simplify the configuration, let's grab the following CA configuration file.

    wget https://raw.githubusercontent.com/anders94/https-authorized-clients/master/keys/ca.cnf  

Next, we'll create a new certificate authority using this configuration.

    openssl req -new -x509 -days 9999 -config ca.cnf -keyout ca-key.pem -out ca-crt.pem  

Now that we have our certificate authority in *ca-key.pem* and *ca-crt.pem*, let's generate a private key for the server.

    openssl genrsa -out server-key.pem 4096  

Our next move is to generate a certificate signing request. Again to simplify configuration, let's use *server.cnf* as a configuration shortcut.

    wget https://raw.githubusercontent.com/anders94/https-authorized-clients/master/keys/server.cnf  

Now we'll generate the certificate signing request.

    openssl req -new -config server.cnf -key server-key.pem -out server-csr.pem  

Now let's sign the request.

    openssl x509 -req -extfile server.cnf -days 999 -passin "pass:password" -in server-csr.pem -CA ca-crt.pem -CAkey ca-key.pem -CAcreateserial -out server-crt.pem  

Our server certificate is all set and ready to go!

Server
------

Next, let's build a basic HTTPS server using the certificate and listen on 0.0.0.0:4433. This is your standard HTTPS server in Node.js.

    var fs = require('fs');  
    var https = require('https');

    var options = {  
        key: fs.readFileSync('server-key.pem'),
        cert: fs.readFileSync('server-crt.pem'),
        ca: fs.readFileSync('ca-crt.pem'),
    };

    https.createServer(options, function (req, res) {  
        console.log(new Date()+' '+
            req.connection.remoteAddress+' '+
            req.method+' '+req.url);
        res.writeHead(200);
        res.end("hello world\n");
      }).listen(4433);

You can check that this works in a browser. (don't forget to tell your operating system to trust our newly created certificate authority by installing *ca-crt.pem* and marking it as trusted)

Client
------

Let's create a Node.js client to connect to the server because that's where we will be demonstrating the client certificate stuff.

    var fs = require('fs');  
    var https = require('https');

    var options = {  
        hostname: 'localhost',
        port: 4433,
        path: '/',
        method: 'GET',
        ca: fs.readFileSync('ca-crt.pem')
    };

    var req = https.request(options, function(res) {  
        res.on('data', function(data) {
            process.stdout.write(data);
          });
      });
    req.end();  

See what we did there? We added a line to the options that sets the certificate authority that the client will trust to our certificate authority's public key. You can test that this works by running it and hopefully seeing "hello world" returned back.

Client Certificates
-------------------

Now, for the certificate on the client side, we're going to need something that the client can present to our new server. Let's make two of them while we're at it.

    openssl genrsa -out client1-key.pem 4096  

    openssl genrsa -out client2-key.pem 4096  

We'll use config files again to save us some typing.

    wget https://raw.githubusercontent.com/anders94/https-authorized-clients/master/keys/client1.cnf  

    wget https://raw.githubusercontent.com/anders94/https-authorized-clients/master/keys/client2.cnf  

Now let's create a pair of certificate signing requests.

    openssl req -new -config client1.cnf -key client1-key.pem -out client1-csr.pem  

    openssl req -new -config client2.cnf -key client2-key.pem -out client2-csr.pem  

And sign our two new client certs.

    openssl x509 -req -extfile client1.cnf -days 999 -passin "pass:password" -in client1-csr.pem -CA ca-crt.pem -CAkey ca-key.pem -CAcreateserial -out client1-crt.pem  

    openssl x509 -req -extfile client2.cnf -days 999 -passin "pass:password" -in client2-csr.pem -CA ca-crt.pem -CAkey ca-key.pem -CAcreateserial -out client2-crt.pem  

Just to make sure everything in the OpenSSL world worked as expected, let's verify our certs. (you can do this with the server certificate as well if you like)

    openssl verify -CAfile ca-crt.pem client1-crt.pem  

    openssl verify -CAfile ca-crt.pem client2-crt.pem  

We should get an "OK" if all is well.

Putting it all Together
-----------------------

Now for the fun part. Let's alter the server by setting requestCert and rejectUnauthorized to true in the options and adding a bit to the console.log line.

    var fs = require('fs');  
    var https = require('https');

    var options = {  
        key: fs.readFileSync('server-key.pem'),
        cert: fs.readFileSync('server-crt.pem'),
        ca: fs.readFileSync('ca-crt.pem'),
        requestCert: true,
        rejectUnauthorized: true
    };

    https.createServer(options, function (req, res) {  
        console.log(new Date()+' '+
            req.connection.remoteAddress+' '+
            req.socket.getPeerCertificate().subject.CN+' '+
            req.method+' '+req.url);
        res.writeHead(200);
        res.end("hello world\n");
      }).listen(4433);

Now if you try to access your HTTPS server, you will probably be rejected because you aren't presenting a valid client certificate. (in Node.js, you will get a socket hangup error because TLS won't complete) Let's add that to the client we were using by adding a key and cert to the options.

    var fs = require('fs');  
    var https = require('https');

    var options = {  
        hostname: 'localhost',
        port: 4433,
        path: '/',
        method: 'GET',
        key: fs.readFileSync('client1-key.pem'),
        cert: fs.readFileSync('client1-crt.pem'),
        ca: fs.readFileSync('ca-crt.pem')
    };

    var req = https.request(options, function(res) {  
        res.on('data', function(data) {
            process.stdout.write(data);
          });
      });
    req.end();

    req.on('error', function(e) {  
        console.error(e);
      });

Now for the moment of truth. Run the above against your server and you should see 'hello world' echo back. Look at the log the server produced and you should see 'client1'. The server has recognized the client and printed it's "Common Name"!

Try changing the client to use the client2 certificate and you should see 'client2' in the server's logging. Cool, eh?

Certificate Revocation
----------------------

There is one more thing to do here. Let's add the ability to revoke a client certificate without invalidating everything. To do this, we will create a Certificate Revocation List (CRL) and revoke the client2 certificate. The first time we'll do this, we need to create an empty database:

    touch ca-database.txt  

Now let's revoke client2's certificate.

    openssl ca -revoke client2-crt.pem -keyfile ca-key.pem -config ca.cnf -cert ca-crt.pem -passin 'pass:password'  

And we'll update the CRL.

    openssl ca -keyfile ca-key.pem -cert ca-crt.pem -config ca.cnf -gencrl -out ca-crl.pem -passin 'pass:password'  

Lastly, we'll have to define **crl** in the options. Our new server will look like this.

    var fs = require('fs');  
    var https = require('https');

    var options = {  
        key: fs.readFileSync('server-key.pem'),
        cert: fs.readFileSync('server-crt.pem'),
        ca: fs.readFileSync('ca-crt.pem'),
        crl: fs.readFileSync('ca-crl.pem'),
        requestCert: true,
        rejectUnauthorized: true
    };

    https.createServer(options, function (req, res) {  
        console.log(new Date()+' '+
            req.connection.remoteAddress+' '+
            req.socket.getPeerCertificate().subject.CN+' '+
            req.method+' '+req.url);
        res.writeHead(200);
        res.end("hello world\n");
      }).listen(4433);

Now our server will respect the certificate revocation list so we should now see client2 rejected while client1 is still accepted!

