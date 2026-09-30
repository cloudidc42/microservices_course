# Part 77: Service Mesh Advanced

## บทนำ

Service Mesh เป็น infrastructure layer ที่จัดการ communication ระหว่าง services บทนี้จะลึกลงไปใน Istio Traffic Management, Envoy Extensions, WebAssembly Filters, mTLS Policies, Telemetry API, External Authorization, Rate Limiting และการ Debug Service Mesh Issues

---

## 1. Istio Traffic Management Deep Dive

### 1.1 Advanced VirtualService Configuration

```yaml
# istio/virtual-service-advanced.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: product-service-vs
  namespace: production
spec:
  hosts:
    - product-service
    - api.company.com
  gateways:
    - production/main-gateway
    - mesh
  http:
    # Canary deployment: 10% traffic to v2
    - name: canary-route
      match:
        - headers:
            x-canary-user:
              exact: "true"
        - queryParams:
            version:
              exact: "v2"
      route:
        - destination:
            host: product-service
            subset: v2
          weight: 100

    # A/B testing based on user segment
    - name: ab-test-route
      match:
        - headers:
            x-user-segment:
              regex: "^(premium|enterprise)$"
      route:
        - destination:
            host: product-service
            subset: v2
          weight: 50
        - destination:
            host: product-service
            subset: v1
          weight: 50

    # Fault injection for testing
    - name: fault-injection-route
      match:
        - headers:
            x-test-fault:
              exact: "delay"
      fault:
        delay:
          percentage:
            value: 100.0
          fixedDelay: 3s
      route:
        - destination:
            host: product-service
            subset: v1

    - name: fault-abort-route
      match:
        - headers:
            x-test-fault:
              exact: "abort"
      fault:
        abort:
          percentage:
            value: 100.0
          httpStatus: 503
      route:
        - destination:
            host: product-service
            subset: v1

    # Traffic mirroring for testing new version
    - name: mirror-route
      match:
        - uri:
            prefix: /api/products
      route:
        - destination:
            host: product-service
            subset: v1
          weight: 100
      mirror:
        host: product-service
        subset: v2
      mirrorPercentage:
        value: 10.0  # Mirror 10% of traffic

    # Default route with retries and timeouts
    - name: default-route
      timeout: 5s
      retries:
        attempts: 3
        perTryTimeout: 2s
        retryOn: "gateway-error,connect-failure,retriable-4xx,503"
        retryRemoteLocalities: true
      route:
        - destination:
            host: product-service
            subset: v1
          weight: 90
        - destination:
            host: product-service
            subset: v2
          weight: 10
      headers:
        request:
          add:
            x-forwarded-service: product-service
          remove:
            - x-internal-token
        response:
          add:
            x-served-by: product-service
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: product-service-dr
  namespace: production
spec:
  host: product-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
        connectTimeout: 30ms
        tcpKeepalive:
          time: 7200s
          interval: 75s
          probes: 10
      http:
        http1MaxPendingRequests: 50
        http2MaxRequests: 1000
        maxRequestsPerConnection: 10
        maxRetries: 5
        idleTimeout: 90s
        h2UpgradePolicy: UPGRADE
    outlierDetection:
      consecutiveGatewayErrors: 5
      consecutiveLocalOriginFailures: 5
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      minHealthPercent: 50
    loadBalancer:
      simple: LEAST_CONN
      localityLbSetting:
        enabled: true
        failoverPriority:
          - "topology.kubernetes.io/zone"
          - "topology.kubernetes.io/region"
  subsets:
    - name: v1
      labels:
        version: v1
      trafficPolicy:
        portLevelSettings:
          - port:
              number: 8080
            connectionPool:
              http:
                http2MaxRequests: 500
    - name: v2
      labels:
        version: v2
      trafficPolicy:
        portLevelSettings:
          - port:
              number: 8080
            connectionPool:
              http:
                http2MaxRequests: 200
```

### 1.2 Gateway Configuration

```yaml
# istio/gateway.yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: main-gateway
  namespace: production
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: SIMPLE
        credentialName: company-tls-secret
        minProtocolVersion: TLSV1_3
        cipherSuites:
          - ECDHE-ECDSA-AES256-GCM-SHA384
          - ECDHE-RSA-AES256-GCM-SHA384
      hosts:
        - "*.company.com"
    - port:
        number: 80
        name: http
        protocol: HTTP
      tls:
        httpsRedirect: true
      hosts:
        - "*.company.com"
    - port:
        number: 9000
        name: tcp-db
        protocol: TCP
      hosts:
        - "db.internal.company.com"
```

---

## 2. Envoy Extensions และ Filters

### 2.1 Envoy HTTP Filter Chain

```yaml
# envoy/envoy-config.yaml
static_resources:
  listeners:
    - name: listener_0
      address:
        socket_address:
          protocol: TCP
          address: 0.0.0.0
          port_value: 10000
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: ingress_http
                use_remote_address: true
                xff_num_trusted_hops: 1
                codec_type: AUTO
                
                # HTTP/2 settings
                http2_protocol_options:
                  max_concurrent_streams: 100
                  initial_stream_window_size: 65536
                  initial_connection_window_size: 1048576

                # Access logging
                access_log:
                  - name: envoy.access_loggers.file
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.access_loggers.file.v3.FileAccessLog
                      path: /dev/stdout
                      log_format:
                        json_format:
                          timestamp: "%START_TIME%"
                          method: "%REQ(:METHOD)%"
                          path: "%REQ(X-ENVOY-ORIGINAL-PATH?:PATH)%"
                          protocol: "%PROTOCOL%"
                          response_code: "%RESPONSE_CODE%"
                          response_flags: "%RESPONSE_FLAGS%"
                          bytes_received: "%BYTES_RECEIVED%"
                          bytes_sent: "%BYTES_SENT%"
                          duration: "%DURATION%"
                          upstream_service_time: "%RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)%"
                          forwarded_for: "%REQ(X-FORWARDED-FOR)%"
                          user_agent: "%REQ(USER-AGENT)%"
                          request_id: "%REQ(X-REQUEST-ID)%"
                          upstream_host: "%UPSTREAM_HOST%"

                # HTTP filters
                http_filters:
                  # Rate limiting
                  - name: envoy.filters.http.local_ratelimit
                    typed_config:
                      "@type": type.googleapis.com/udpa.type.v1.TypedStruct
                      type_url: type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
                      value:
                        stat_prefix: http_local_rate_limiter
                        token_bucket:
                          max_tokens: 1000
                          tokens_per_fill: 1000
                          fill_interval: 1s
                        filter_enabled:
                          runtime_key: local_rate_limit_enabled
                          default_value:
                            numerator: 100
                            denominator: HUNDRED
                        filter_enforced:
                          runtime_key: local_rate_limit_enforced
                          default_value:
                            numerator: 100
                            denominator: HUNDRED

                  # JWT authentication
                  - name: envoy.filters.http.jwt_authn
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.jwt_authn.v3.JwtAuthentication
                      providers:
                        company_auth:
                          issuer: "https://auth.company.com"
                          audiences:
                            - "api.company.com"
                          remote_jwks:
                            http_uri:
                              uri: "https://auth.company.com/.well-known/jwks.json"
                              cluster: auth_cluster
                              timeout: 5s
                            cache_duration:
                              seconds: 300
                          forward: true
                          forward_payload_header: x-jwt-payload
                      rules:
                        - match:
                            prefix: /api/
                          requires:
                            provider_name: company_auth
                        - match:
                            prefix: /health
                          allow_missing_or_failed: {}

                  # External authorization
                  - name: envoy.filters.http.ext_authz
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.ext_authz.v3.ExtAuthz
                      grpc_service:
                        envoy_grpc:
                          cluster_name: ext_authz_cluster
                        timeout: 0.5s
                      transport_api_version: V3
                      include_peer_certificate: true
                      failure_mode_allow: false

                  # Lua filter for custom logic
                  - name: envoy.filters.http.lua
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.lua.v3.LuaPerRoute
                      inline_code: |
                        function envoy_on_request(request_handle)
                          local headers = request_handle:headers()
                          local request_id = headers:get("x-request-id")
                          if request_id == nil then
                            request_handle:headers():add("x-request-id", 
                              string.format("%08x-%04x-4%03x-%04x-%12x",
                                math.random(0xFFFFFFFF),
                                math.random(0xFFFF),
                                math.random(0x0FFF),
                                math.random(0x3FFF) + 0x8000,
                                math.random(0xFFFFFFFFFFFF)))
                          end
                        end

                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
```

---

## 3. WebAssembly Filters

### 3.1 Custom WASM Filter ด้วย Go

```go
// wasm/rate-limit-filter/main.go
package main

import (
	"github.com/tetratelabs/proxy-wasm-go-sdk/proxywasm"
	"github.com/tetratelabs/proxy-wasm-go-sdk/proxywasm/types"
	"strconv"
	"strings"
	"time"
)

func main() {
	proxywasm.SetVMContext(&vmContext{})
}

type vmContext struct{}

func (*vmContext) OnVMStart(vmConfigurationSize int) types.OnVMStartStatus {
	return types.OnVMStartStatusOK
}

func (*vmContext) NewPluginContext(contextID uint32) types.PluginContext {
	return &pluginContext{}
}

type pluginContext struct {
	types.DefaultPluginContext
	maxRequests int
	windowSec   int
}

func (p *pluginContext) OnPluginStart(pluginConfigurationSize int) types.OnPluginStartStatus {
	data, err := proxywasm.GetPluginConfiguration()
	if err != nil {
		return types.OnPluginStartStatusFailed
	}
	
	config := string(data)
	for _, line := range strings.Split(config, "\n") {
		parts := strings.SplitN(line, "=", 2)
		if len(parts) != 2 {
			continue
		}
		switch strings.TrimSpace(parts[0]) {
		case "max_requests":
			p.maxRequests, _ = strconv.Atoi(strings.TrimSpace(parts[1]))
		case "window_seconds":
			p.windowSec, _ = strconv.Atoi(strings.TrimSpace(parts[1]))
		}
	}

	if p.maxRequests == 0 {
		p.maxRequests = 100
	}
	if p.windowSec == 0 {
		p.windowSec = 60
	}

	return types.OnPluginStartStatusOK
}

func (p *pluginContext) NewHttpContext(contextID uint32) types.HttpContext {
	return &httpContext{
		contextID:   contextID,
		maxRequests: p.maxRequests,
		windowSec:   p.windowSec,
	}
}

type httpContext struct {
	types.DefaultHttpContext
	contextID   uint32
	maxRequests int
	windowSec   int
}

func (h *httpContext) OnHttpRequestHeaders(numHeaders int, endOfStream bool) types.Action {
	// Get client IP
	clientIP, err := proxywasm.GetHttpRequestHeader(":authority")
	if err != nil {
		return types.ActionContinue
	}

	// Check rate limit using shared data
	windowKey := strconv.FormatInt(time.Now().Unix()/int64(h.windowSec), 10)
	key := "ratelimit:" + clientIP + ":" + windowKey

	cas := uint32(0)
	data, err := proxywasm.GetSharedData(key)
	var count int

	if err == nil {
		count, _ = strconv.Atoi(string(data))
	}

	count++

	if err := proxywasm.SetSharedData(key, []byte(strconv.Itoa(count)), cas); err != nil {
		return types.ActionContinue
	}

	if count > h.maxRequests {
		_ = proxywasm.SendHttpResponse(429, [][2]string{
			{"content-type", "application/json"},
			{"x-ratelimit-limit", strconv.Itoa(h.maxRequests)},
			{"x-ratelimit-remaining", "0"},
			{"retry-after", strconv.Itoa(h.windowSec)},
		}, []byte(`{"error":"rate limit exceeded"}`), -1)
		return types.ActionPause
	}

	remaining := h.maxRequests - count
	_ = proxywasm.AddHttpRequestHeader("x-ratelimit-remaining", strconv.Itoa(remaining))
	return types.ActionContinue
}
```

### 3.2 Deploy WASM Filter

```yaml
# kubernetes/wasm-plugin.yaml
apiVersion: extensions.istio.io/v1alpha1
kind: WasmPlugin
metadata:
  name: custom-rate-limiter
  namespace: production
spec:
  selector:
    matchLabels:
      app: product-service
  url: oci://registry.company.com/wasm/rate-limiter:v1.0.0
  phase: AUTHN
  priority: 100
  pluginConfig:
    max_requests: "200"
    window_seconds: "60"
  imagePullSecret: registry-credentials
  imagePullPolicy: IfNotPresent
```

---

## 4. mTLS Policy Types

### 4.1 Peer Authentication Policies

```yaml
# istio/mtls-policies.yaml
# Strict mTLS for entire namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: namespace-wide-mtls
  namespace: production
spec:
  mtls:
    mode: STRICT
---
# Per-service mTLS with port exceptions
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: product-service-mtls
  namespace: production
spec:
  selector:
    matchLabels:
      app: product-service
  mtls:
    mode: STRICT
  portLevelMtls:
    # Allow plaintext on metrics port (for Prometheus scraping outside mesh)
    9090:
      mode: DISABLE
    # Permissive during migration
    8081:
      mode: PERMISSIVE
---
# Request Authentication (JWT)
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-auth
  namespace: production
spec:
  selector:
    matchLabels:
      app: product-service
  jwtRules:
    - issuer: "https://auth.company.com"
      jwksUri: "https://auth.company.com/.well-known/jwks.json"
      audiences:
        - "api.company.com"
      forwardOriginalToken: true
      outputClaimToHeaders:
        - header: x-user-id
          claim: sub
        - header: x-user-roles
          claim: roles
---
# Authorization Policy
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: product-service-authz
  namespace: production
spec:
  selector:
    matchLabels:
      app: product-service
  action: ALLOW
  rules:
    # Allow internal services (mTLS verified)
    - from:
        - source:
            principals:
              - "cluster.local/ns/production/sa/order-service"
              - "cluster.local/ns/production/sa/cart-service"
      to:
        - operation:
            methods: ["GET", "POST"]
            paths: ["/api/products*"]

    # Allow external traffic through ingress gateway
    - from:
        - source:
            principals:
              - "cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account"
      to:
        - operation:
            methods: ["GET"]
            paths: ["/api/products*"]
      when:
        - key: request.auth.claims[roles]
          values: ["user", "admin", "premium"]

    # Allow health checks from kubelet
    - to:
        - operation:
            methods: ["GET"]
            paths: ["/health*", "/metrics"]
---
# Deny all traffic by default (explicit whitelist)
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all-default
  namespace: production
spec:
  {}  # Empty spec = deny all
```

---

## 5. Telemetry API

### 5.1 Custom Metrics และ Tracing

```yaml
# istio/telemetry.yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: custom-metrics
  namespace: production
spec:
  selector:
    matchLabels:
      app: product-service
  metrics:
    - providers:
        - name: prometheus
      overrides:
        # Add custom tags to metrics
        - match:
            mode: CLIENT_AND_SERVER
          tagOverrides:
            user_segment:
              value: request.headers['x-user-segment'] | "unknown"
            feature_flag:
              value: request.headers['x-feature-flag'] | "none"
            cache_status:
              value: response.headers['x-cache'] | "MISS"
        # Disable high-cardinality metric to save resources
        - match:
            metric: REQUEST_SIZE
          disabled: true
  tracing:
    - providers:
        - name: jaeger
      customTags:
        user_id:
          header:
            name: x-user-id
        session_id:
          header:
            name: x-session-id
        service_version:
          environment:
            name: SERVICE_VERSION
            defaultValue: "unknown"
      randomSamplingPercentage: 10.0
  accessLogging:
    - providers:
        - name: otel
      filter:
        expression: "response.code >= 400"  # Only log errors
```

### 5.2 Prometheus Custom Metrics

```yaml
# istio/prometheus-metrics.yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: business-metrics
  namespace: production
spec:
  metrics:
    - providers:
        - name: prometheus
      overrides:
        - match:
            mode: SERVER
          tagOverrides:
            # Business-level dimensions
            product_category:
              value: |
                has(request.url_path) && request.url_path.contains("/products/") ?
                  request.headers['x-product-category'] | "unknown" :
                  "n/a"
            payment_method:
              value: |
                request.headers['x-payment-method'] | "unknown"
            ab_variant:
              value: |
                request.headers['x-ab-variant'] | "control"
```

---

## 6. External Authorization

### 6.1 External Authorization Server

```go
// ext-authz/server.go
package main

import (
	"context"
	"log"
	"net"

	core "github.com/envoyproxy/go-control-plane/envoy/config/core/v3"
	auth "github.com/envoyproxy/go-control-plane/envoy/service/auth/v3"
	envoy_type "github.com/envoyproxy/go-control-plane/envoy/type/v3"
	"github.com/golang/protobuf/ptypes/wrappers"
	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	status "google.golang.org/grpc/status"
)

type AuthorizationServer struct {
	tokenStore TokenStore
	policyEngine PolicyEngine
}

func (s *AuthorizationServer) Check(
	ctx context.Context,
	req *auth.CheckRequest,
) (*auth.CheckResponse, error) {
	httpReq := req.Attributes.Request.Http
	headers := httpReq.Headers

	// Extract JWT token
	authHeader := headers["authorization"]
	if authHeader == "" {
		return denyWithStatus(401, "Missing authorization header"), nil
	}

	token := authHeader
	if len(authHeader) > 7 && authHeader[:7] == "Bearer " {
		token = authHeader[7:]
	}

	// Validate token
	claims, err := s.tokenStore.ValidateToken(token)
	if err != nil {
		return denyWithStatus(401, "Invalid token"), nil
	}

	// Check policy
	decision, err := s.policyEngine.Evaluate(PolicyInput{
		UserID:     claims.Subject,
		UserRoles:  claims.Roles,
		Method:     httpReq.Method,
		Path:       httpReq.Path,
		RemoteAddr: req.Attributes.Source.Address.GetSocketAddress().GetAddress(),
		Headers:    headers,
	})

	if err != nil || !decision.Allow {
		reason := "Access denied"
		if decision != nil {
			reason = decision.Reason
		}
		return denyWithStatus(403, reason), nil
	}

	// Return approval with additional headers
	return &auth.CheckResponse{
		Status: &status.Status{Code: int32(codes.OK)},
		HttpResponse: &auth.CheckResponse_OkResponse{
			OkResponse: &auth.OkHttpResponse{
				Headers: []*core.HeaderValueOption{
					{
						Header: &core.HeaderValue{
							Key:   "x-user-id",
							Value: claims.Subject,
						},
						Append: &wrappers.BoolValue{Value: false},
					},
					{
						Header: &core.HeaderValue{
							Key:   "x-user-roles",
							Value: joinRoles(claims.Roles),
						},
						Append: &wrappers.BoolValue{Value: false},
					},
					{
						Header: &core.HeaderValue{
							Key:   "x-auth-decision",
							Value: "allowed",
						},
						Append: &wrappers.BoolValue{Value: false},
					},
				},
				HeadersToRemove: []string{"authorization"},
			},
		},
	}, nil
}

func denyWithStatus(code int32, body string) *auth.CheckResponse {
	return &auth.CheckResponse{
		Status: &status.Status{Code: int32(codes.PermissionDenied)},
		HttpResponse: &auth.CheckResponse_DeniedResponse{
			DeniedResponse: &auth.DeniedHttpResponse{
				Status: &envoy_type.HttpStatus{
					Code: envoy_type.StatusCode(code),
				},
				Headers: []*core.HeaderValueOption{
					{
						Header: &core.HeaderValue{
							Key:   "content-type",
							Value: "application/json",
						},
					},
				},
				Body: `{"error":"` + body + `"}`,
			},
		},
	}
}

func main() {
	lis, err := net.Listen("tcp", ":9001")
	if err != nil {
		log.Fatalf("Failed to listen: %v", err)
	}

	grpcServer := grpc.NewServer(
		grpc.MaxRecvMsgSize(1024*1024),
		grpc.MaxSendMsgSize(1024*1024),
	)
	
	server := &AuthorizationServer{
		tokenStore:   NewRedisTokenStore(),
		policyEngine: NewOPAPolicyEngine("./policies"),
	}

	auth.RegisterAuthorizationServer(grpcServer, server)
	log.Printf("External authz server listening on :9001")
	grpcServer.Serve(lis)
}

func joinRoles(roles []string) string {
	result := ""
	for i, r := range roles {
		if i > 0 {
			result += ","
		}
		result += r
	}
	return result
}
```

---

## 7. Rate Limiting via Envoy

### 7.1 Global Rate Limiting Service

```go
// rate-limit-service/main.go
package main

import (
	"context"
	"log"
	"net"
	"sync"
	"time"

	pb "github.com/envoyproxy/go-control-plane/envoy/service/ratelimit/v3"
	"google.golang.org/grpc"
)

type TokenBucket struct {
	tokens    float64
	maxTokens float64
	refillRate float64  // tokens per second
	lastRefill time.Time
	mu         sync.Mutex
}

func (tb *TokenBucket) Allow() bool {
	tb.mu.Lock()
	defer tb.mu.Unlock()
	
	now := time.Now()
	elapsed := now.Sub(tb.lastRefill).Seconds()
	tb.tokens = min(tb.maxTokens, tb.tokens + elapsed * tb.refillRate)
	tb.lastRefill = now
	
	if tb.tokens >= 1 {
		tb.tokens--
		return true
	}
	return false
}

type RateLimitServer struct {
	buckets map[string]*TokenBucket
	mu      sync.RWMutex
	limits  map[string]RateLimit
}

type RateLimit struct {
	RequestsPerUnit int
	Unit            string
}

var defaultLimits = map[string]RateLimit{
	"global_ip":           {RequestsPerUnit: 1000, Unit: "MINUTE"},
	"user_api":            {RequestsPerUnit: 200, Unit: "MINUTE"},
	"auth_login":          {RequestsPerUnit: 10, Unit: "MINUTE"},
	"payment_api":         {RequestsPerUnit: 20, Unit: "MINUTE"},
	"search_api":          {RequestsPerUnit: 60, Unit: "MINUTE"},
	"export_api":          {RequestsPerUnit: 5, Unit: "HOUR"},
}

func (s *RateLimitServer) ShouldRateLimit(
	ctx context.Context,
	req *pb.RateLimitRequest,
) (*pb.RateLimitResponse, error) {
	response := &pb.RateLimitResponse{
		Statuses: make([]*pb.RateLimitResponse_DescriptorStatus, len(req.Descriptors)),
	}

	for i, descriptor := range req.Descriptors {
		key := s.buildKey(req.Domain, descriptor)
		allowed := s.checkLimit(key)

		code := pb.RateLimitResponse_OK
		if !allowed {
			code = pb.RateLimitResponse_OVER_LIMIT
		}

		limit := s.getLimit(key)
		response.Statuses[i] = &pb.RateLimitResponse_DescriptorStatus{
			Code: code,
			CurrentLimit: &pb.RateLimitResponse_RateLimit{
				RequestsPerUnit: uint32(limit.RequestsPerUnit),
				Unit:            pb.RateLimitResponse_RateLimit_Unit(pb.RateLimitResponse_RateLimit_Unit_value[limit.Unit]),
			},
			LimitRemaining: s.getRemaining(key),
		}
	}

	return response, nil
}

func (s *RateLimitServer) buildKey(domain string, desc *pb.RateLimitDescriptor) string {
	key := domain
	for _, entry := range desc.Entries {
		key += ":" + entry.Key + ":" + entry.Value
	}
	return key
}

func (s *RateLimitServer) checkLimit(key string) bool {
	s.mu.Lock()
	bucket, exists := s.buckets[key]
	if !exists {
		limit := s.getLimit(key)
		tokensPerSecond := float64(limit.RequestsPerUnit)
		switch limit.Unit {
		case "MINUTE":
			tokensPerSecond /= 60.0
		case "HOUR":
			tokensPerSecond /= 3600.0
		case "DAY":
			tokensPerSecond /= 86400.0
		}
		bucket = &TokenBucket{
			tokens:     float64(limit.RequestsPerUnit),
			maxTokens:  float64(limit.RequestsPerUnit),
			refillRate: tokensPerSecond,
			lastRefill: time.Now(),
		}
		s.buckets[key] = bucket
	}
	s.mu.Unlock()
	return bucket.Allow()
}

func (s *RateLimitServer) getLimit(key string) RateLimit {
	for limitKey, limit := range s.limits {
		if containsKey(key, limitKey) {
			return limit
		}
	}
	return RateLimit{RequestsPerUnit: 1000, Unit: "MINUTE"}
}

func (s *RateLimitServer) getRemaining(key string) uint32 {
	s.mu.RLock()
	defer s.mu.RUnlock()
	bucket, exists := s.buckets[key]
	if !exists {
		return 0
	}
	return uint32(bucket.tokens)
}

func containsKey(key, pattern string) bool {
	return len(key) >= len(pattern) && key[:len(pattern)] == pattern
}

func min(a, b float64) float64 {
	if a < b {
		return a
	}
	return b
}

func main() {
	lis, err := net.Listen("tcp", ":8081")
	if err != nil {
		log.Fatalf("Failed to listen: %v", err)
	}

	server := &RateLimitServer{
		buckets: make(map[string]*TokenBucket),
		limits:  defaultLimits,
	}

	grpcServer := grpc.NewServer()
	pb.RegisterRateLimitServiceServer(grpcServer, server)
	log.Println("Rate limit service listening on :8081")
	grpcServer.Serve(lis)
}
```

---

## 8. Debugging Service Mesh Issues

### 8.1 Debug Scripts

```bash
#!/bin/bash
# scripts/debug-service-mesh.sh

SERVICE_NAME=${1:-"product-service"}
NAMESPACE=${2:-"production"}
POD=$(kubectl get pod -n $NAMESPACE -l app=$SERVICE_NAME -o jsonpath='{.items[0].metadata.name}')

echo "=== Debugging Service Mesh for: $SERVICE_NAME ==="
echo "Pod: $POD"
echo ""

# 1. Check Envoy proxy status
echo "--- Envoy Proxy Status ---"
kubectl exec -n $NAMESPACE $POD -c istio-proxy -- pilot-agent request GET server_info

echo ""
echo "--- Envoy Clusters Status ---"
kubectl exec -n $NAMESPACE $POD -c istio-proxy -- pilot-agent request GET clusters | \
  python3 -c "import sys,json; data=sys.stdin.read(); print(data)" 2>/dev/null || \
  kubectl exec -n $NAMESPACE $POD -c istio-proxy -- curl -s localhost:15000/clusters

echo ""
echo "--- Envoy Listeners ---"
kubectl exec -n $NAMESPACE $POD -c istio-proxy -- pilot-agent request GET listeners

echo ""
echo "--- Envoy Config Dump ---"
kubectl exec -n $NAMESPACE $POD -c istio-proxy -- \
  curl -s localhost:15000/config_dump | \
  python3 -c "
import sys, json
data = json.load(sys.stdin)
for config in data.get('configs', []):
    name = config.get('@type', 'unknown').split('.')[-1]
    print(f'Config type: {name}')
" 2>/dev/null

echo ""
echo "--- mTLS Status ---"
istioctl authn tls-check $POD.$NAMESPACE 2>/dev/null || \
  echo "istioctl not available, checking manually..."
kubectl exec -n $NAMESPACE $POD -c istio-proxy -- \
  curl -s localhost:15000/config_dump | \
  python3 -c "
import sys, json
data = json.load(sys.stdin)
# Find TLS contexts
" 2>/dev/null

echo ""
echo "--- Authorization Policies ---"
kubectl get authorizationpolicy -n $NAMESPACE

echo ""
echo "--- Recent Errors in Proxy Logs ---"
kubectl logs $POD -n $NAMESPACE -c istio-proxy --tail=50 | grep -E "error|warn|ERR|WARN" 2>/dev/null

echo ""
echo "--- Envoy Stats ---"
kubectl exec -n $NAMESPACE $POD -c istio-proxy -- \
  curl -s localhost:15000/stats | grep -E "upstream_rq_5xx|upstream_rq_time|upstream_cx_connect_fail"
```

### 8.2 Service Mesh Diagnostic Tool

```typescript
// src/tools/mesh-diagnostics.ts
import { exec } from 'child_process';
import { promisify } from 'util';

const execAsync = promisify(exec);

interface ServiceStatus {
  service: string;
  namespace: string;
  pods: PodStatus[];
  virtualService?: any;
  destinationRule?: any;
  authPolicy?: any;
  mtlsMode?: string;
}

interface PodStatus {
  name: string;
  ready: boolean;
  proxyStatus: string;
  warnings: string[];
}

export class MeshDiagnostics {
  async diagnoseService(service: string, namespace: string): Promise<ServiceStatus> {
    const [pods, vs, dr, authPolicy] = await Promise.allSettled([
      this.getPodStatuses(service, namespace),
      this.getVirtualService(service, namespace),
      this.getDestinationRule(service, namespace),
      this.getAuthPolicy(service, namespace),
    ]);

    const mtlsMode = await this.checkMTLSMode(service, namespace);

    return {
      service,
      namespace,
      pods: pods.status === 'fulfilled' ? pods.value : [],
      virtualService: vs.status === 'fulfilled' ? vs.value : null,
      destinationRule: dr.status === 'fulfilled' ? dr.value : null,
      authPolicy: authPolicy.status === 'fulfilled' ? authPolicy.value : null,
      mtlsMode,
    };
  }

  private async getPodStatuses(service: string, namespace: string): Promise<PodStatus[]> {
    const { stdout } = await execAsync(
      `kubectl get pods -n ${namespace} -l app=${service} -o json`
    );
    const data = JSON.parse(stdout);

    return data.items.map((pod: any) => {
      const containers = pod.status.containerStatuses || [];
      const proxyContainer = containers.find((c: any) => c.name === 'istio-proxy');
      const warnings: string[] = [];

      if (!proxyContainer?.ready) {
        warnings.push('Istio proxy is not ready');
      }

      return {
        name: pod.metadata.name,
        ready: pod.status.phase === 'Running',
        proxyStatus: proxyContainer?.state?.running ? 'Running' : 'Not Running',
        warnings,
      };
    });
  }

  private async getVirtualService(service: string, namespace: string): Promise<any> {
    try {
      const { stdout } = await execAsync(
        `kubectl get virtualservice -n ${namespace} -l app=${service} -o json 2>/dev/null || echo "{}"`
      );
      return JSON.parse(stdout);
    } catch {
      return null;
    }
  }

  private async getDestinationRule(service: string, namespace: string): Promise<any> {
    try {
      const { stdout } = await execAsync(
        `kubectl get destinationrule ${service} -n ${namespace} -o json 2>/dev/null || echo "{}"`
      );
      return JSON.parse(stdout);
    } catch {
      return null;
    }
  }

  private async getAuthPolicy(service: string, namespace: string): Promise<any> {
    try {
      const { stdout } = await execAsync(
        `kubectl get authorizationpolicy -n ${namespace} -l app=${service} -o json 2>/dev/null || echo "{}"`
      );
      return JSON.parse(stdout);
    } catch {
      return null;
    }
  }

  private async checkMTLSMode(service: string, namespace: string): Promise<string> {
    try {
      const { stdout } = await execAsync(
        `kubectl get peerauthentication -n ${namespace} -o json 2>/dev/null || echo "{}"`
      );
      const data = JSON.parse(stdout);
      const items = data.items || [];
      
      // Check for service-specific policy
      const servicePolicy = items.find((item: any) =>
        item.spec?.selector?.matchLabels?.app === service
      );
      
      if (servicePolicy) {
        return servicePolicy.spec?.mtls?.mode || 'PERMISSIVE';
      }
      
      // Check namespace-wide policy
      const namespacePolicy = items.find((item: any) => !item.spec?.selector);
      return namespacePolicy?.spec?.mtls?.mode || 'PERMISSIVE';
    } catch {
      return 'UNKNOWN';
    }
  }

  async generateReport(service: string, namespace: string): Promise<string> {
    const status = await this.diagnoseService(service, namespace);
    
    let report = `# Service Mesh Diagnostic Report\n\n`;
    report += `**Service:** ${status.service}\n`;
    report += `**Namespace:** ${status.namespace}\n`;
    report += `**mTLS Mode:** ${status.mtlsMode}\n\n`;

    report += `## Pods\n`;
    for (const pod of status.pods) {
      const statusIcon = pod.ready ? '✓' : '✗';
      report += `- ${statusIcon} ${pod.name} (Proxy: ${pod.proxyStatus})\n`;
      for (const warning of pod.warnings) {
        report += `  ⚠️  ${warning}\n`;
      }
    }

    if (status.virtualService) {
      report += `\n## VirtualService\n`;
      report += `Routes configured: ${status.virtualService.items?.length || 0}\n`;
    }

    if (status.authPolicy) {
      report += `\n## Authorization Policy\n`;
      report += `Policies found: ${status.authPolicy.items?.length || 0}\n`;
    }

    return report;
  }
}
```

---

## 9. Observability Stack Integration

### 9.1 Kiali Dashboard Configuration

```yaml
# kubernetes/kiali-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kiali
  namespace: istio-system
data:
  config.yaml: |
    auth:
      openid:
        client_id: kiali
        issuer_uri: "https://auth.company.com"
        username_claim: email
      strategy: openid

    deployment:
      accessible_namespaces:
        - production
        - staging
      
    external_services:
      prometheus:
        url: "http://prometheus-server.monitoring:9090"
      grafana:
        enabled: true
        in_cluster_url: "http://grafana.monitoring:3000"
        url: "https://grafana.company.com"
      jaeger:
        in_cluster_url: "http://jaeger-query.monitoring:16686"
        url: "https://jaeger.company.com"

    api:
      namespaces:
        exclude:
          - istio-system
          - kube-system
    
    kiali_feature_flags:
      certificates_information_indicators:
        enabled: true
        secrets:
          - cacerts
          - istio-ca-secret
```

---

## สรุป

บทนี้ครอบคลุม Service Mesh Advanced Patterns อย่างละเอียด:

1. **Istio Traffic Management** - VirtualService ขั้นสูง, Canary deployments, Fault injection, Traffic mirroring
2. **Envoy Configuration** - HTTP filter chains, JWT auth, Rate limiting, Access logging
3. **WebAssembly Filters** - Custom Go WASM filter สำหรับ rate limiting ที่ edge
4. **mTLS Policies** - PeerAuthentication, RequestAuthentication, AuthorizationPolicy
5. **Telemetry API** - Custom metrics tags, Distributed tracing, Access log filtering
6. **External Authorization** - gRPC-based authz server ด้วย Go
7. **Rate Limiting** - Token bucket algorithm ใน dedicated service
8. **Debugging Tools** - Scripts และ TypeScript tools สำหรับ diagnose mesh issues

Key Takeaways:
- Traffic mirroring ช่วยทดสอบ new version โดยไม่กระทบ production traffic
- WASM filters ให้ extensibility ที่ high performance ไม่ต้องแก้ Envoy source
- mTLS STRICT mode ควรใช้ใน production ทุก namespace ที่ไม่มี external traffic
- External Authorization แยก auth logic ออกจาก application layer
- ใช้ outlierDetection เสมอเพื่อ circuit breaking อัตโนมัติ
