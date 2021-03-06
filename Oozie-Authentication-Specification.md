## Oozie Authentication Specification

The goal of this document is to provide developer a tutorial of how to write your own authentication and configure in the run-time environment of a Oozie server.

### 1. Oozie Authentication Definitions

   **Authenticator**: A client side class to authenticate user and send the authentication information to server along with each request.

   **AuthenticationProvider**: A server side component to retrieve authentication token from http request and validate the token.

   **AuthenticationToken**: A object contains authentication information for a request.

### 2. Oozie Authentication Introduction

Oozie Authentication provides a framework to let developer provide a custom implementation to authenticate the requests from Oozie client. The client side authentication code is used to send the authentication information as a header in the HTTP request sent to the server. It can be used to send different kinds of authentication tokens or certificates. After a successful authentication using one of the configured methods, it sends Hadoop-HTTP-Auth cookie in further requests.

The server side authentication module has a AuthenticationProviderFactory which needs to be initialized with the required AuthenticationProviders from a configuration file. The authentication is handled by the AuthenticationProcessingFilter. Once a request is received the following happens in the filter:

   * Request is checked for presence of Hadoop-HTTP-Auth cookie. If present the signature is verified and the username is extracted from the cookie.
   * If the cookie is not present or is invalid, a supported provider is fetched from the AuthenticationProviderFactory passing in the Request. If there is no supported provider a 401 is sent with the header "WWW-Authenticate: Negotiate"
   * If a supported provider is found, the authentication is delegated to the supported provider. On successful 
authentication, the provider returns an instance of AuthenticationToken which contains the authenticated user.
   * An UGI object is constructed out of the authenticated user and set as the request attribute "authorized.ugi" which can be later consumed by the servlets. 

### 3. Server Authentication Implementation

To write a new custom authentication for server, two classes have to be provided with overrided implementation. 

**1. Provider:** three methods are required to implement.    
    * supports() : the method checks if its authentication mechanism supports the authentication information a request provided.
    * getAuthenticationToken() : the method is called after supports() returns true and used to constructs the token from parameters in a request. 
    * authenticate() : the method is used to validate a token created above and rewrite it with new information if needed.

**2. Token:** the instance contains the information for a authentication provider to use.

For example,

_SimpleAuthenticationHeadProvider_ implements the AuthenticationProvider to check if client has send a parameter (username) in the request.

_SimpleAuthenticationToken_ extends AbstractAuthenticationToken to set the authentication flag to true and save client parameter in a instance of Token.

### 4. Client Authentication Implementation

To write a new custom authentication for client, one class have to be provided with overrided implementation. 

*HttpAuthenticator:* one methods is required to implement.    
    * authenticate(conf, connection) : the method is used to do the client authentication and insert authentication information to http connection.

For example,

SimpleAuthenticator extends the HttpAuthenticator and send user name as http connection's request parameter.

### 5. Server Authentication Configuration

An authentication provider can be given in 'oozie-site.xml' for Oozie server. The property 'authentication.providers' is used to configure what authentication mechanisms are supported in Oozie server runtime.

```xml
  <property>
	<name>authentication.providers</name>
	<value>org.apache.hadoop.http.authentication.server.simple.SimpleAuthenticationHeaderProvider</value>
	<description>Comma separated list of authentication providers in FQCN.</description>
  </property>
```

### 6. Client Authentication Configuration

To use an authenticator in client, a configuration "oozie.auth.map" has to be defined in client-site.xml or other user-specified client site xml. The value of "oozie.auth.map" specifies the mapping from key “XXX” to “org.apache.XXXauthenticator”. The format of "oozie.auth.map" is like "key1=value1,key2=value2". A key “XXX” is used at command line to specify which authenticator to use.

For example,

Default client-site.xml in Oozie Client can be found in oozie-client tar.

```xml
<configuration>

    <!--  Oozie client authenticator -->
    <property>
        <name>oozie.auth.map</name>
        <value>simple=org.apache.hadoop.http.authentication.client.simple.SimpleAuthenticator</value>
        <description>
            List of Oozie client authenticator (separated by commas).
            The format is key=classname and key will be used in command line for -auth options.
        </description>
    </property>

</configuration>
```

* Command line option "-clientsite" or environment variable "CLIENT_SITE" is used to specify the client site xml.

### 7 Client Authentication CommandLine

In command line, argument "-auth" has to be given to use user-defined authenticator. Default is SimpleAuthenticator if "-auth" is not given. A command line sample is:
```text
         Ex. oozie job –run –config map-red.properties –auth XXX -clientsite PATH_TO_CLIENT_SITE -Dkey1=value –Dkey2=value
```
The value of "-clientsite" or environment variable "CLIENT_SITE" is the path to the client site xml which contains client site configuration. The value of "-auth" is a key in the value "oozie.auth.map" of client site xml. When "-auth" is used, Oozie client looks for "-clientsite", or env variable "CLIENT_SITE" if "-clientsite" is not present.

For example,

To use simple authenticator from above example,
```bash
$ oozie job –run –config map-red.properties –auth simple -clientsite PATH_TO_CLIENT_SITE
```