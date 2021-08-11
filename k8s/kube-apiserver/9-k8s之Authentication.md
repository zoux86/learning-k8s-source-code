Table of Contents
=================

  * [1. 简介](#1-简介)
  * [2. 认证器的生成](#2-认证器的生成)
     * [2.1 调用链路](#21-调用链路)
     * [2.2 BuildAuthenticator](#22-buildauthenticator)
     * [2.3 ToAuthenticationConfig](#23-toauthenticationconfig)
     * [2.4 New](#24-new)
  * [3. 具体的认证过程](#3-具体的认证过程)
     * [3.1 调用链路](#31-调用链路)
     * [3.2 t.handler到底是谁](#32-thandler到底是谁)
     * [3.3 DefaultBuildHandlerChain](#33-defaultbuildhandlerchain)
     * [3.4 WithAuthentication](#34-withauthentication)
  * [4. 9种认证方式介绍](#4-9种认证方式介绍)
     * [4.1 BasicAuth认证](#41-basicauth认证)
     * [4.2 ClientCA认证](#42-clientca认证)
     * [4.3 TokenAuth认证](#43-tokenauth认证)
     * [4.4 BootstrapToken认证](#44-bootstraptoken认证)
     * [4.5  RequestHeader认证](#45--requestheader认证)
     * [4.6 WebhookTokenAuth认证](#46-webhooktokenauth认证)
     * [4.7 Anonymous认证](#47-anonymous认证)
     * [4.8 OIDC认证](#48-oidc认证)
     * [4.9 ServiceAccountAuth认证](#49-serviceaccountauth认证)
     * [4.10 总结](#410-总结)
  * [5.参考链接：](#5参考链接)

### 1. 简介

kube-apiserver作为一个服务器端。每次请求到来时都需要经过认证授权，以及一系列的访问控制。k8s1.17版本中，共提供9中认证方式：Anonymous，BootstrapToken，ClientCert，OIDC，PasswordFile，RequestHeader，ServiceAccounts，TokenFile，WebHook。

认证和授权的区别： 

假设apiserver收到了一个请求，一个名叫张三的用户想删除 namespaceA下的一个pod。

**认证：** apiserver判断你到底是不是张三

**授权：** 张三到底有没有删除这个pod的权限

<br>

### 2. 认证器的生成

#### 2.1 调用链路

run函数经过一系列的调用，最终调用BuildAuthenticator函数来生成认证器。

cmd/kube-apiserver/app/server.go Run ->  CreateServerChain ->  CreateKubeAPIServerConfig -> buildGenericConfig ->BuildAuthenticator 

```
以下函数只显示关键的代码
// Run runs the specified APIServer.  This should never exit.
func Run(completeOptions completedServerRunOptions, stopCh <-chan struct{}) error {
	server, err := CreateServerChain(completeOptions, stopCh)
}


// CreateServerChain creates the apiservers connected via delegation.
func CreateServerChain(completedOptions completedServerRunOptions, stopCh <-chan struct{}) (*aggregatorapiserver.APIAggregator, error) {


	kubeAPIServerConfig, insecureServingInfo, serviceResolver, pluginInitializer, err := CreateKubeAPIServerConfig(completedOptions, nodeTunneler, proxyTransport)
	
}


// CreateKubeAPIServerConfig creates all the resources for running the API server, but runs none of them
func CreateKubeAPIServerConfig(
	s completedServerRunOptions,
	nodeTunneler tunneler.Tunneler,
	proxyTransport *http.Transport,
) (
	*master.Config,
	*genericapiserver.DeprecatedInsecureServingInfo,
	aggregatorapiserver.ServiceResolver,
	[]admission.PluginInitializer,
	error,
) {

	genericConfig, versionedInformers, insecureServingInfo, serviceResolver, pluginInitializers, admissionPostStartHook, storageFactory, err := buildGenericConfig(s.ServerRunOptions, proxyTransport)

}


// BuildGenericConfig takes the master server options and produces the genericapiserver.Config associated with it
func buildGenericConfig(
	s *options.ServerRunOptions,
	proxyTransport *http.Transport,
) (
	genericConfig *genericapiserver.Config,
	versionedInformers clientgoinformers.SharedInformerFactory,
	insecureServingInfo *genericapiserver.DeprecatedInsecureServingInfo,
	serviceResolver aggregatorapiserver.ServiceResolver,
	pluginInitializers []admission.PluginInitializer,
	admissionPostStartHook genericapiserver.PostStartHookFunc,
	storageFactory *serverstorage.DefaultStorageFactory,
	lastErr error,
) {
	
  // 认证
	genericConfig.Authentication.Authenticator, genericConfig.OpenAPIConfig.SecurityDefinitions, err = BuildAuthenticator(s, clientgoExternalClient, versionedInformers)
	if err != nil {
		lastErr = fmt.Errorf("invalid authentication config: %v", err)
		return
	}

  //授权
	genericConfig.Authorization.Authorizer, genericConfig.RuleResolver, err =BuildAuthorizer(s, versionedInformers)
}
```

<br>

####  2.2 BuildAuthenticator

BuildAuthenticator目前是关键的函数，从这里开始进行代码分析

```
// BuildAuthenticator constructs the authenticator
func BuildAuthenticator(s *options.ServerRunOptions, extclient clientgoclientset.Interface, versionedInformer clientgoinformers.SharedInformerFactory) (authenticator.Request, *spec.SecurityDefinitions, error) {
   
   // 1.生成config
   authenticatorConfig, err := s.Authentication.ToAuthenticationConfig()
   if err != nil {
      return nil, nil, err
   }
   if s.Authentication.ServiceAccounts.Lookup || utilfeature.DefaultFeatureGate.Enabled(features.TokenRequest) {
      authenticatorConfig.ServiceAccountTokenGetter = serviceaccountcontroller.NewGetterFromClient(
         extclient,
         versionedInformer.Core().V1().Secrets().Lister(),
         versionedInformer.Core().V1().ServiceAccounts().Lister(),
         versionedInformer.Core().V1().Pods().Lister(),
      )
   }
   authenticatorConfig.BootstrapTokenAuthenticator = bootstrap.NewTokenAuthenticator(
      versionedInformer.Core().V1().Secrets().Lister().Secrets(v1.NamespaceSystem),
   )
   
   // 2.更加config，生成每个认证方式的 handler
   return authenticatorConfig.New()
}
```

<br>

#### 2.3 ToAuthenticationConfig

可以看出来这里ToAuthenticationConfig就是根据输入的配置，判断哪些认证方式（上面说的九种）需要生成config

这里直接看代码函数就行。

<br>

#### 2.4 New

（1）New函数根据认证的配置信息，针对9中认证方法，生成对应的handler。具体做法就是将各种认证生成authenticator，加入authenticators数组

（2）将authenticators数组生成一个union handler

（3）最终得认证器AuthenticatedGroupAdder

```
// New returns an authenticator.Request or an error that supports the standard
// Kubernetes authentication mechanisms.
func (config Config) New() (authenticator.Request, *spec.SecurityDefinitions, error) {
	var authenticators []authenticator.Request
	var tokenAuthenticators []authenticator.Token
	securityDefinitions := spec.SecurityDefinitions{}

	// front-proxy, BasicAuth methods, local first, then remote
	// Add the front proxy authenticator if requested
	if config.RequestHeaderConfig != nil {
		requestHeaderAuthenticator := headerrequest.NewDynamicVerifyOptionsSecure(
			config.RequestHeaderConfig.CAContentProvider.VerifyOptions,
			config.RequestHeaderConfig.AllowedClientNames,
			config.RequestHeaderConfig.UsernameHeaders,
			config.RequestHeaderConfig.GroupHeaders,
			config.RequestHeaderConfig.ExtraHeaderPrefixes,
		)
		authenticators = append(authenticators, authenticator.WrapAudienceAgnosticRequest(config.APIAudiences, requestHeaderAuthenticator))
	}
  
  // 1.将各种认证生成authenticator，加入authenticators数组
	// basic auth
	if len(config.BasicAuthFile) > 0 {
		basicAuth, err := newAuthenticatorFromBasicAuthFile(config.BasicAuthFile)
		if err != nil {
			return nil, nil, err
		}
		authenticators = append(authenticators, authenticator.WrapAudienceAgnosticRequest(config.APIAudiences, basicAuth))

		securityDefinitions["HTTPBasic"] = &spec.SecurityScheme{
			SecuritySchemeProps: spec.SecuritySchemeProps{
				Type:        "basic",
				Description: "HTTP Basic authentication",
			},
		}
	}

	// X509 methods
	if config.ClientCAContentProvider != nil {
		certAuth := x509.NewDynamic(config.ClientCAContentProvider.VerifyOptions, x509.CommonNameUserConversion)
		authenticators = append(authenticators, certAuth)
	}

	// Bearer token methods, local first, then remote
	if len(config.TokenAuthFile) > 0 {
		tokenAuth, err := newAuthenticatorFromTokenFile(config.TokenAuthFile)
		if err != nil {
			return nil, nil, err
		}
		tokenAuthenticators = append(tokenAuthenticators, authenticator.WrapAudienceAgnosticToken(config.APIAudiences, tokenAuth))
	}
	if len(config.ServiceAccountKeyFiles) > 0 {
		serviceAccountAuth, err := newLegacyServiceAccountAuthenticator(config.ServiceAccountKeyFiles, config.ServiceAccountLookup, config.APIAudiences, config.ServiceAccountTokenGetter)
		if err != nil {
			return nil, nil, err
		}
		tokenAuthenticators = append(tokenAuthenticators, serviceAccountAuth)
	}
	if utilfeature.DefaultFeatureGate.Enabled(features.TokenRequest) && config.ServiceAccountIssuer != "" {
		serviceAccountAuth, err := newServiceAccountAuthenticator(config.ServiceAccountIssuer, config.ServiceAccountKeyFiles, config.APIAudiences, config.ServiceAccountTokenGetter)
		if err != nil {
			return nil, nil, err
		}
		tokenAuthenticators = append(tokenAuthenticators, serviceAccountAuth)
	}
	if config.BootstrapToken {
		if config.BootstrapTokenAuthenticator != nil {
			// TODO: This can sometimes be nil because of
			tokenAuthenticators = append(tokenAuthenticators, authenticator.WrapAudienceAgnosticToken(config.APIAudiences, config.BootstrapTokenAuthenticator))
		}
	}
	// NOTE(ericchiang): Keep the OpenID Connect after Service Accounts.
	//
	// Because both plugins verify JWTs whichever comes first in the union experiences
	// cache misses for all requests using the other. While the service account plugin
	// simply returns an error, the OpenID Connect plugin may query the provider to
	// update the keys, causing performance hits.
	if len(config.OIDCIssuerURL) > 0 && len(config.OIDCClientID) > 0 {
		oidcAuth, err := newAuthenticatorFromOIDCIssuerURL(oidc.Options{
			IssuerURL:            config.OIDCIssuerURL,
			ClientID:             config.OIDCClientID,
			APIAudiences:         config.APIAudiences,
			CAFile:               config.OIDCCAFile,
			UsernameClaim:        config.OIDCUsernameClaim,
			UsernamePrefix:       config.OIDCUsernamePrefix,
			GroupsClaim:          config.OIDCGroupsClaim,
			GroupsPrefix:         config.OIDCGroupsPrefix,
			SupportedSigningAlgs: config.OIDCSigningAlgs,
			RequiredClaims:       config.OIDCRequiredClaims,
		})
		if err != nil {
			return nil, nil, err
		}
		tokenAuthenticators = append(tokenAuthenticators, oidcAuth)
	}
	if len(config.WebhookTokenAuthnConfigFile) > 0 {
		webhookTokenAuth, err := newWebhookTokenAuthenticator(config.WebhookTokenAuthnConfigFile, config.WebhookTokenAuthnVersion, config.WebhookTokenAuthnCacheTTL, config.APIAudiences)
		if err != nil {
			return nil, nil, err
		}
		tokenAuthenticators = append(tokenAuthenticators, webhookTokenAuth)
	}

	if len(tokenAuthenticators) > 0 {
		// Union the token authenticators
		tokenAuth := tokenunion.New(tokenAuthenticators...)
		// Optionally cache authentication results
		if config.TokenSuccessCacheTTL > 0 || config.TokenFailureCacheTTL > 0 {
			tokenAuth = tokencache.New(tokenAuth, true, config.TokenSuccessCacheTTL, config.TokenFailureCacheTTL)
		}
		authenticators = append(authenticators, bearertoken.New(tokenAuth), websocket.NewProtocolAuthenticator(tokenAuth))
		securityDefinitions["BearerToken"] = &spec.SecurityScheme{
			SecuritySchemeProps: spec.SecuritySchemeProps{
				Type:        "apiKey",
				Name:        "authorization",
				In:          "header",
				Description: "Bearer Token authentication",
			},
		}
	}

	if len(authenticators) == 0 {
		if config.Anonymous {
			return anonymous.NewAuthenticator(), &securityDefinitions, nil
		}
		return nil, &securityDefinitions, nil
	}

  // 2. 生成一个union handler
	authenticator := union.New(authenticators...)
 
 
  // 3.最终得认证器AuthenticatedGroupAdder
	authenticator = group.NewAuthenticatedGroupAdder(authenticator)

	if config.Anonymous {
		// If the authenticator chain returns an error, return an error (don't consider a bad bearer token
		// or invalid username/password combination anonymous).
		authenticator = union.NewFailOnError(authenticator, anonymous.NewAuthenticator())
	}

	return authenticator, &securityDefinitions, nil
}


// union.New函数。
// New returns a request authenticator that validates credentials using a chain of authenticator.Request objects.
// The entire chain is tried until one succeeds. If all fail, an aggregate error is returned.
func New(authRequestHandlers ...authenticator.Request) authenticator.Request {
	if len(authRequestHandlers) == 1 {
		return authRequestHandlers[0]
	}
	return &unionAuthRequestHandler{Handlers: authRequestHandlers, FailOnError: false}
}
```

为什么要弄成一个unionAuthRequestHandler，原因在于unionAuthRequestHandler有一个这样的函数AuthenticateRequest。

从这里可以看出来，unionAuthRequestHandler分别调用各种认证方法的handler，如果有一种方法认证成功，则成功，返回相应的用户信息。

```
// AuthenticateRequest authenticates the request using a chain of authenticator.Request objects.
func (authHandler *unionAuthRequestHandler) AuthenticateRequest(req *http.Request) (*authenticator.Response, bool, error) {
	var errlist []error
	for _, currAuthRequestHandler := range authHandler.Handlers {
		resp, ok, err := currAuthRequestHandler.AuthenticateRequest(req)
		if err != nil {
			if authHandler.FailOnError {
				return resp, ok, err
			}
			errlist = append(errlist, err)
			continue
		}

		if ok {
			return resp, ok, err
		}
	}

	return nil, false, utilerrors.NewAggregate(errlist)
}
```

<br>

### 3. 具体的认证过程

Authenticator步骤的输入是整个HTTP请求，但是，它通常只是检查HTTP Headers and/or client certificate。

可以指定多个Authenticator模块，在这种情况下，每个认证模块都按顺序尝试，直到其中一个成功即可。

如果认证成功，则用户的`username`会传入授权模块做进一步授权验证；而对于认证失败的请求则返回HTTP 401。

Kubernetes使用client certificates, bearer tokens, an authenticating proxy, or HTTP basic auth, 通过身份验证插件对API请求进行身份验证。 当向API服务器发出一个HTTP请求，Authentication plugin会尝试将以下属性与请求关联：

- Username: 标识终端用户的字符串, 常用值可能是kube-admin或[jane@example.com](mailto:jane@example.com)。
- UID: 标识终端用户的字符串,比Username更具有唯一性。
- Groups: a set of strings which associate users with a set of commonly grouped users.
- Extra fields: 可能有用的额外信息

系统中把这4个属性封装成一个type DefaultInfo struct ，见/pkg/auth/user/user.go。

```
// DefaultInfo provides a simple user information exchange object
// for components that implement the UserInfo interface.
type DefaultInfo struct {
	Name   string
	UID    string
	Groups []string
	Extra  map[string][]string
}
```



#### 3.1 调用链路

staging/src/k8s.io/apiserver/pkg/server/filters/timeout.go ServeHTTP -> ServeHTTP -> 

```
func (t *timeoutHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
  ...
   go func() {
      defer func() {
         err := recover()
         // do not wrap the sentinel ErrAbortHandler panic value
         if err != nil && err != http.ErrAbortHandler {
            // Same as stdlib http server code. Manually allocate stack
            // trace buffer size to prevent excessively large logs
            const size = 64 << 10
            buf := make([]byte, size)
            buf = buf[:runtime.Stack(buf, false)]
            err = fmt.Sprintf("%v\n%s", err, buf)
         }
         resultCh <- err
      }()
      t.handler.ServeHTTP(tw, r)
   }()
}

ServeHTTP是一个接口，所以就看t.handler是谁
// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```

<br>

#### 3.2 t.handler到底是谁

staging/src/k8s.io/apiserver/pkg/server/config.go

Apiserver的config中定义了handler函数。这是一串链式handler函数。

```
// NewConfig returns a Config struct with the default values
func NewConfig(codecs serializer.CodecFactory) *Config {
	defaultHealthChecks := []healthz.HealthChecker{healthz.PingHealthz, healthz.LogHealthz}
	return &Config{
		Serializer:                  codecs,
		BuildHandlerChainFunc:       DefaultBuildHandlerChain,
}
```

<br>

#### 3.3 DefaultBuildHandlerChain

DefaultBuildHandlerChain定义了链式handle。其中认证的就是 WithAuthentication函数

```
func DefaultBuildHandlerChain(apiHandler http.Handler, c *Config) http.Handler {
	handler := genericapifilters.WithAuthorization(apiHandler, c.Authorization.Authorizer, c.Serializer)
	handler = genericfilters.WithMaxInFlightLimit(handler, c.MaxRequestsInFlight, c.MaxMutatingRequestsInFlight, c.LongRunningFunc)
	handler = genericapifilters.WithImpersonation(handler, c.Authorization.Authorizer, c.Serializer)
	handler = genericapifilters.WithAudit(handler, c.AuditBackend, c.AuditPolicyChecker, c.LongRunningFunc)
	failedHandler := genericapifilters.Unauthorized(c.Serializer, c.Authentication.SupportsBasicAuth)
	// 认证的handler
	failedHandler = genericapifilters.WithFailedAuthenticationAudit(failedHandler, c.AuditBackend, c.AuditPolicyChecker)
	handler = genericapifilters.WithAuthentication(handler, c.Authentication.Authenticator, failedHandler, c.Authentication.APIAudiences)
	handler = genericfilters.WithCORS(handler, c.CorsAllowedOriginList, nil, nil, nil, "true")
	handler = genericfilters.WithTimeoutForNonLongRunningRequests(handler, c.LongRunningFunc, c.RequestTimeout)
	handler = genericfilters.WithWaitGroup(handler, c.LongRunningFunc, c.HandlerChainWaitGroup)
	handler = genericapifilters.WithRequestInfo(handler, c.RequestInfoResolver)
	handler = genericfilters.WithPanicRecovery(handler)
	return handler
}
```

<br>

#### 3.4 WithAuthentication

WithAuthentication主要干了两件事：

（1）调用AuthenticateRequest进行了认证。这里实际就是之前的unionAuthRequestHandler.AuthenticateRequest

unionAuthRequestHandler.AuthenticateRequest会遍历所有的认证handler，然后有一个认证成功，就返回ok。

（2）如果认证失败，调用failed.ServeHTTP(w, req)进行处理

（3）如果成功， req.Header.Del("Authorization")删除头部的Authorization, 表示认证通过了

```
// WithAuthentication creates an http handler that tries to authenticate the given request as a user, and then
// stores any such user found onto the provided context for the request. If authentication fails or returns an error
// the failed handler is used. On success, "Authorization" header is removed from the request and handler
// is invoked to serve the request.
func WithAuthentication(handler http.Handler, auth authenticator.Request, failed http.Handler, apiAuds authenticator.Audiences) http.Handler {
   if auth == nil {
      klog.Warningf("Authentication is disabled")
      return handler
   }
   return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
      authenticationStart := time.Now()

      if len(apiAuds) > 0 {
         req = req.WithContext(authenticator.WithAudiences(req.Context(), apiAuds))
      }
      // 这里调用了AuthenticateRequest进行认证
      resp, ok, err := auth.AuthenticateRequest(req)
      if err != nil || !ok {
         if err != nil {
            klog.Errorf("Unable to authenticate the request due to an error: %v", err)
            authenticatedAttemptsCounter.WithLabelValues(errorLabel).Inc()
            authenticationLatency.WithLabelValues(errorLabel).Observe(time.Since(authenticationStart).Seconds())
         } else if !ok {
            authenticatedAttemptsCounter.WithLabelValues(failureLabel).Inc()
            authenticationLatency.WithLabelValues(failureLabel).Observe(time.Since(authenticationStart).Seconds())
         }

         failed.ServeHTTP(w, req)
         return
      }

      if len(apiAuds) > 0 && len(resp.Audiences) > 0 && len(authenticator.Audiences(apiAuds).Intersect(resp.Audiences)) == 0 {
         klog.Errorf("Unable to match the audience: %v , accepted: %v", resp.Audiences, apiAuds)
         failed.ServeHTTP(w, req)
         return
      }

      // authorization header is not required anymore in case of a successful authentication.
      req.Header.Del("Authorization")

      req = req.WithContext(genericapirequest.WithUser(req.Context(), resp.User))

      authenticatedUserCounter.WithLabelValues(compressUsername(resp.User.GetName())).Inc()
      authenticatedAttemptsCounter.WithLabelValues(successLabel).Inc()
      authenticationLatency.WithLabelValues(successLabel).Observe(time.Since(authenticationStart).Seconds())

      handler.ServeHTTP(w, req)
   })
}
```

### 4. 9种认证方式介绍

#### 4.1 BasicAuth认证

BasicAuth是一种简单的HTTP协议上的认证机制，客户端将用户、密码写入请求头中，HTTP服务端尝试从请求头中验证用户、密码信息，从而实现身份验证。客户端发送的请求头示例如下：

```
Authorization: Basic BASE64ENCODED(USER:PASSWORD)
```

请求头的key为Authorization，value为Basic BASE64ENCODED（USER：PASSWORD），其中用户名及密码是通过Base64编码后的字符串。

<br>

**启用BasicAuth认证:**

kube-apiserver通过指定--basic-auth-file参数启用BasicAuth认证。AUTH_FILE是一个CSV文件，每个用户在CSV中的表现形式为password、username、uid，代码示例如下：

```
a0d175cf548f665938498,derk,1 
```

<br>

**认证函数:**

staging/src/k8s.io/apiserver/plugin/pkg/authenticator/request/basicauth/basicauth.go

```
// AuthenticateRequest authenticates the request using the "Authorization: Basic" header in the request
func (a *Authenticator) AuthenticateRequest(req *http.Request) (*authenticator.Response, bool, error) {
	username, password, found := req.BasicAuth()
	if !found {
		return nil, false, nil
	}

	resp, ok, err := a.auth.AuthenticatePassword(req.Context(), username, password)

	// If the password authenticator didn't error, provide a default error
	if !ok && err == nil {
		err = errInvalidAuth
	}

	return resp, ok, err
}
```

<br>

#### 4.2 ClientCA认证

ClientCA认证，也被称为TLS双向认证，即服务端与客户端互相验证证书的正确性。使用ClientCA认证的时候，只要是CA签名过的证书都可以通过验证。1.启用ClientCA认证kube-apiserver通过指定--client-ca-file参数启用ClientCA认证。这个目前比较常用。

ClientCA认证接口定义了AuthenticateRequest方法，该方法接收客户端请求。若验证失败，bool值会为false；若验证成功，bool值会为true，并返回*authenticator.Response，*authenticator.Response中携带了身份验证用户的信息，例如Name、UID、Groups、Extra等信息。

staging/src/k8s.io/apiserver/pkg/authentication/request/x509/x509.go

```
// AuthenticateRequest authenticates the request using presented client certificates
func (a *Authenticator) AuthenticateRequest(req *http.Request) (*authenticator.Response, bool, error) {
	if req.TLS == nil || len(req.TLS.PeerCertificates) == 0 {
		return nil, false, nil
	}

	// Use intermediates, if provided
	optsCopy, ok := a.verifyOptionsFn()
	// if there are intentionally no verify options, then we cannot authenticate this request
	if !ok {
		return nil, false, nil
	}
	if optsCopy.Intermediates == nil && len(req.TLS.PeerCertificates) > 1 {
		optsCopy.Intermediates = x509.NewCertPool()
		for _, intermediate := range req.TLS.PeerCertificates[1:] {
			optsCopy.Intermediates.AddCert(intermediate)
		}
	}

	remaining := req.TLS.PeerCertificates[0].NotAfter.Sub(time.Now())
	clientCertificateExpirationHistogram.Observe(remaining.Seconds())
	chains, err := req.TLS.PeerCertificates[0].Verify(optsCopy)
	if err != nil {
		return nil, false, err
	}

	var errlist []error
	for _, chain := range chains {
		user, ok, err := a.user.User(chain)
		if err != nil {
			errlist = append(errlist, err)
			continue
		}

		if ok {
			return user, ok, err
		}
	}
	return nil, false, utilerrors.NewAggregate(errlist)
}
```

在进行ClientCA认证时，通过req.TLS.PeerCertifcates[0].Verify验证证书，如果是CA签名过的证书，都可以通过验证，认证失败会返回false，而认证成功会返回true。

<br>

#### 4.3 TokenAuth认证

Token也被称为令牌，服务端为了验证客户端的身份，需要客户端向服务端提供一个可靠的验证信息，这个验证信息就是Token。TokenAuth是基于Token的认证，Token一般是一个字符串。

**启用TokenAuth认证**

kube-apiserver通过指定--token-auth-file参数启用TokenAuth认证。TOKEN_FILE是一个CSV文件，每个用户在CSV中的表现形式为token、user、userid、group，代码示例如下：

```
a0d73844190894384102943,kubelet-bootstrap.1001,"system:kubelet-bootstrap"
```

Token认证接口定义了AuthenticateToken方法，该方法接收token字符串。若验证失败，bool值会为false；若验证成功，bool值会为true，并返回*authenticator.Response，*authenticator.Response中携带了身份验证用户的信息，例如Name、UID、Groups、Extra等信息。

```
func (a *TokenAuthenticator) AuthenticateToken(ctx context.Context, value string) (*authenticator.Response, bool, error) {
	user, ok := a.tokens[value]
	if !ok {
		return nil, false, nil
	}
	return &authenticator.Response{User: user}, true, nil
}
```

<br>

#### 4.4 BootstrapToken认证

当Kubernetes集群中有非常多的节点时，手动为每个节点配置TLS认证比较烦琐，为此Kubernetes提供了BootstrapToken认证，其也被称为引导Token。客户端的Token信息与服务端的Token相匹配，则认证通过，自动为节点颁发证书，这是一种引导Token的机制。客户端发送的请求头示例如下：

```
Authorization: Bearer 07410b.f2355rejewrql
```

请求头的key为Authorization，value为Bearer<TOKENS>，其中TOKENS的表现形式为[a-z0-9]{6}.[a-z0-9]{16}。第一个组是Token ID，第二个组是TokenSecret。

**启用BootstrapToken认证**

kube-apiserver通过指定--enable-bootstrap-token-auth参数启用BootstrapToken认证。

这个在安装kubelet的时候使用过。

BootstrapToken认证接口定义了AuthenticateToken方法，该方法接收token字符串。若验证失败，bool值会为false；若验证成功，bool值会为true，并返回*authenticator.Response，*authenticator.Response中携带了身份验证用户的信息，例如Name、UID、Groups、Extra等信息。

plugin/pkg/auth/authenticator/token/bootstrap/bootstrap.go

```
func (t *TokenAuthenticator) AuthenticateToken(ctx context.Context, token string) (*authenticator.Response, bool, error) {
	tokenID, tokenSecret, err := bootstraptokenutil.ParseToken(token)
	if err != nil {
		// Token isn't of the correct form, ignore it.
		return nil, false, nil
	}

	secretName := bootstrapapi.BootstrapTokenSecretPrefix + tokenID
	secret, err := t.lister.Get(secretName)
	if err != nil {
		if errors.IsNotFound(err) {
			klog.V(3).Infof("No secret of name %s to match bootstrap bearer token", secretName)
			return nil, false, nil
		}
		return nil, false, err
	}

	if secret.DeletionTimestamp != nil {
		tokenErrorf(secret, "is deleted and awaiting removal")
		return nil, false, nil
	}

	if string(secret.Type) != string(bootstrapapi.SecretTypeBootstrapToken) || secret.Data == nil {
		tokenErrorf(secret, "has invalid type, expected %s.", bootstrapapi.SecretTypeBootstrapToken)
		return nil, false, nil
	}

	ts := bootstrapsecretutil.GetData(secret, bootstrapapi.BootstrapTokenSecretKey)
	if subtle.ConstantTimeCompare([]byte(ts), []byte(tokenSecret)) != 1 {
		tokenErrorf(secret, "has invalid value for key %s, expected %s.", bootstrapapi.BootstrapTokenSecretKey, tokenSecret)
		return nil, false, nil
	}

	id := bootstrapsecretutil.GetData(secret, bootstrapapi.BootstrapTokenIDKey)
	if id != tokenID {
		tokenErrorf(secret, "has invalid value for key %s, expected %s.", bootstrapapi.BootstrapTokenIDKey, tokenID)
		return nil, false, nil
	}

	if bootstrapsecretutil.HasExpired(secret, time.Now()) {
		// logging done in isSecretExpired method.
		return nil, false, nil
	}

	if bootstrapsecretutil.GetData(secret, bootstrapapi.BootstrapTokenUsageAuthentication) != "true" {
		tokenErrorf(secret, "not marked %s=true.", bootstrapapi.BootstrapTokenUsageAuthentication)
		return nil, false, nil
	}

	groups, err := bootstrapsecretutil.GetGroups(secret)
	if err != nil {
		tokenErrorf(secret, "has invalid value for key %s: %v.", bootstrapapi.BootstrapTokenExtraGroupsKey, err)
		return nil, false, nil
	}

	return &authenticator.Response{
		User: &user.DefaultInfo{
			Name:   bootstrapapi.BootstrapUserPrefix + string(id),
			Groups: groups,
		},
	}, true, nil
}

```

在进行BootstrapToken认证时，通过paseToken函数解析出Token ID和TokenSecret，验证Token Secret中的Expire（过期）、Data、Type等，认证失败会返回false，而认证成功会返回true。

#### 4.5  RequestHeader认证

Kubernetes可以设置一个认证代理，客户端发送的认证请求可以通过认证代理将验证信息发送给kube-apiserver组件。RequestHeader认证使用的就是这种代理方式，它使用请求头将用户名和组信息发送给kube-apiserver。

RequestHeader认证有几个列表，分别介绍如下。

● 用户名列表。建议使用X-Remote-User，如果启用RequestHeader认证，该参数必选。

● 组列表。建议使用X-Remote-Group，如果启用RequestHeader认证，该参数可选。

● 额外列表。建议使用X-Remote-Extra-，如果启用RequestHeader认证，该参数可选。

当客户端发送认证请求时，kube-apiserver根据Header Values中的用户名列表来识别用户，例如返回X-Remote-User：Bob则表示验证成功。

**启用RequestHeader认证**

kube-apiserver通过指定如下参数启用RequestHeader认证。

●--requestheader-client-ca-file：指定有效的客户端CA证书。

●--requestheader-allowed-names：指定通用名称（CommonName）。

●--requestheader-extra-headers-prefix：指定额外列表。

●--requestheader-group-headers：指定组列表。

●--requestheader-username-headers：指定用户名列表。

kube-apiserver收到客户端验证请求后，会先通过--requestheader-client-ca-file参数对客户端证书进行验证。

--requestheader-username-headers参数指定了Header中包含的用户名，这一参数中的列表确定了有效的用户名列表，如果该列表为空，则所有通过--requestheader-client-ca-file参数校验的请求都允许通过。

```
func (a *requestHeaderAuthRequestHandler) AuthenticateRequest(req *http.Request) (*authenticator.Response, bool, error) {
	name := headerValue(req.Header, a.nameHeaders.Value())
	if len(name) == 0 {
		return nil, false, nil
	}
	groups := allHeaderValues(req.Header, a.groupHeaders.Value())
	extra := newExtra(req.Header, a.extraHeaderPrefixes.Value())

	// clear headers used for authentication
	for _, headerName := range a.nameHeaders.Value() {
		req.Header.Del(headerName)
	}
	for _, headerName := range a.groupHeaders.Value() {
		req.Header.Del(headerName)
	}
	for k := range extra {
		for _, prefix := range a.extraHeaderPrefixes.Value() {
			req.Header.Del(prefix + k)
		}
	}

	return &authenticator.Response{
		User: &user.DefaultInfo{
			Name:   name,
			Groups: groups,
			Extra:  extra,
		},
	}, true, nil
}
```

在进行RequestHeader认证时，通过headerValue函数从请求头中读取所有的用户信息，通过allHeaderValues函数读取所有组的信息，通过newExtra函数读取所有额外的信息。当用户名无法匹配时，则认证失败返回false，反之则认证成功返回true。

<br>

#### 4.6 WebhookTokenAuth认证

Webhook也被称为钩子，是一种基于HTTP协议的回调机制，当客户端发送的认证请求到达kube-apiserver时，kube-apiserver回调钩子方法，将验证信息发送给远程的Webhook服务器进行认证，然后根据Webhook服务器返回的状态码来判断是否认证成功。

**启用WebhookTokenAuth认证**

kube-apiserver通过指定如下参数启用WebhookTokenAuth认证。

●--authentication-token-webhook-config-file：Webhook配置文件描述了如何访问远程Webhook服务。

●--authentication-token-webhook-cache-ttl：缓存认证时间，默认值为2分钟。

<br>

WebhookTokenAuth认证接口定义了AuthenticateToken方法，该方法接收token字符串。若验证失败，bool值会为false；若验证成功，bool值会为true，并返回*authenticator.Response，*authenticator.Response中携带了身份验证用户的信息，例如Name、UID、Groups、Extra等信息。

<br>

#### 4.7 Anonymous认证

Anonymous认证就是匿名认证，未被其他认证器拒绝的请求都可视为匿名请求。kube-apiserver默认开启Anonymous（匿名）认证。1.启用Anonymous认证kube-apiserver通过指定--anonymous-auth参数启用Anonymous认证，默认该参数值为true。

Anonymous认证接口定义了AuthenticateRequest方法，该方法接收客户端请求。若验证失败，bool值会为false；若验证成功，bool值会为true，并返回*authenticator.Response，*authenticator.Response中携带了身份验证用户的信息，例如Name、UID、Groups、Extra等信息。

在进行Anonymous认证时，直接验证成功，返回true。

#### 4.8 OIDC认证

OIDC（OpenID Connect）是一套基于OAuth 2.0协议的轻量级认证规范，其提供了通过API进行身份交互的框架。OIDC认证除了认证请求外，还会标明请求的用户身份（ID Token）。其中Toekn被称为ID Token，此ID Token是JSON WebToken （JWT），具有由服务器签名的相关字段。

OIDC认证流程介绍如下。（1）Kubernetes用户想访问Kubernetes API Server，先通过认证服务（AuthServer，例如Google Accounts服务）认证自己，得到access_token、id_token和refresh_token。（2）Kubernetes用户把access_token、id_token和refresh_token配置到客户端应用程序（如kubectl或dashboard工具等）中。（3）Kubernetes客户端使用Token以用户的身份访问Kubernetes API Server。Kubernetes API Server和Auth Server并没有直接进行交互，而是鉴定客户端发送的Token是否为合法Token。

**启用OIDC认证**

kube-apiserver通过指定如下参数启用OIDC认证。

●--oidc-ca-file：签署身份提供商的CA证书的路径，默认值为主机的根CA证书的路径（即/etc/kubernetes/ssl/kc-ca.pem）。

●--oidc-client-id：颁发所有Token的Client ID。

●--oidc-groups-claim：JWT（JSON Web Token）声明的用户组名称。

●--oidc-groups-prefix：组名前缀，所有组都将以此值为前缀，以避免与其他身份验证策略发生冲突。

●--oidc-issuer-url：Auth Server服务的URL地址，例如使用GoogleAccounts服务。

●--oidc-required-claim：该参数是键值对，用于描述ID Token中的必要声明。如果设置该参数，则验证声明是否以匹配值存在于ID Token中。重复指定该参数可以设置多个声明。

●--oidc-signing-algs：JOSE非对称签名算法列表，算法以逗号分隔。如果以alg开头的JWT请求不在此列表中，请求会被拒绝（默认值为[RS256]）。

●--oidc-username-claim：JWT（JSON Web Token）声明的用户名称（默认值为sub）。●--oidc-username-prefix：用户名前缀，所有用户名都将以此值为前缀，以避免与其他身份验证策略发生冲突。如果要跳过任何前缀，请设置该参数值为-。

#### 4.9 ServiceAccountAuth认证

ServiceAccountAuth是一种特殊的认证机制，其他认证机制都是处于Kubernetes集群外部而希望访问kube-apiserver组件，而ServiceAccountAuth认证是从Pod资源内部访问kube-apiserver组件，提供给运行在Pod资源中的进程使用，它为Pod资源中的进程提供必要的身份证明，从而获取集群的信息。ServiceAccountAuth认证通过Kubernetes资源的Service Account实现。

具体使用就是在创建pod的时候，定义使用ServiceAccount。

#### 4.10 总结

这一部分基本都是摘抄kubernetes源码解剖部分的内容。目的就是先了解一下具体有哪些认证，以后有需要再深入了解一下。

<br>

### 5.参考链接：

https://www.jianshu.com/p/daa4ff387a78

书籍：kubernetes源码解剖，郑东
