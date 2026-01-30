# Filter Chain

## Overview

Servlet Filters in Spring Boot operate at a lower level than interceptors, working directly with the servlet container. They are part of the Java Servlet specification and allow you to intercept and manipulate HTTP requests and responses before they reach the Spring MVC layer.

## Key Concepts

### Filter vs Interceptor

| Feature | Filter | Interceptor |
|---------|--------|-------------|
| Level | Servlet Container | Spring MVC |
| Access | Request/Response | Handler, ModelAndView |
| Exception Handling | Limited | Full @ExceptionHandler support |
| Spring Context | Available but limited | Full access |
| Order | @Order or FilterRegistrationBean | InterceptorRegistry order |

### Filter Interface

```java
public interface Filter {
    void init(FilterConfig filterConfig);
    void doFilter(ServletRequest request, ServletResponse response, FilterChain chain);
    void destroy();
}
```

## Implementation Examples

### Basic Logging Filter

```java
@Component
@Order(1)
public class RequestLoggingFilter implements Filter {
    
    private static final Logger logger = LoggerFactory.getLogger(RequestLoggingFilter.class);
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) 
            throws IOException, ServletException {
        
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;
        
        logger.info("Request: {} {}", httpRequest.getMethod(), httpRequest.getRequestURI());
        
        long startTime = System.currentTimeMillis();
        
        chain.doFilter(request, response);
        
        long duration = System.currentTimeMillis() - startTime;
        logger.info("Response: {} - {} ms", httpResponse.getStatus(), duration);
    }
}
```

### CORS Filter

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class CorsFilter implements Filter {
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) 
            throws IOException, ServletException {
        
        HttpServletResponse httpResponse = (HttpServletResponse) response;
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        
        httpResponse.setHeader("Access-Control-Allow-Origin", "*");
        httpResponse.setHeader("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS");
        httpResponse.setHeader("Access-Control-Allow-Headers", "Authorization, Content-Type");
        httpResponse.setHeader("Access-Control-Max-Age", "3600");
        
        if ("OPTIONS".equalsIgnoreCase(httpRequest.getMethod())) {
            httpResponse.setStatus(HttpServletResponse.SC_OK);
        } else {
            chain.doFilter(request, response);
        }
    }
}
```

### Request/Response Wrapper Filter

```java
@Component
@Order(2)
public class RequestResponseLoggingFilter implements Filter {
    
    private static final Logger logger = LoggerFactory.getLogger(RequestResponseLoggingFilter.class);
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) 
            throws IOException, ServletException {
        
        ContentCachingRequestWrapper requestWrapper = new ContentCachingRequestWrapper((HttpServletRequest) request);
        ContentCachingResponseWrapper responseWrapper = new ContentCachingResponseWrapper((HttpServletResponse) response);
        
        chain.doFilter(requestWrapper, responseWrapper);
        
        String requestBody = new String(requestWrapper.getContentAsByteArray(), requestWrapper.getCharacterEncoding());
        String responseBody = new String(responseWrapper.getContentAsByteArray(), responseWrapper.getCharacterEncoding());
        
        logger.info("Request Body: {}", requestBody);
        logger.info("Response Body: {}", responseBody);
        
        responseWrapper.copyBodyToResponse();
    }
}
```

### Authentication Filter

```java
@Component
@Order(3)
public class JwtAuthenticationFilter implements Filter {
    
    @Autowired
    private JwtTokenProvider tokenProvider;
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) 
            throws IOException, ServletException {
        
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;
        
        String token = extractToken(httpRequest);
        
        if (token != null && tokenProvider.validateToken(token)) {
            String username = tokenProvider.getUsernameFromToken(token);
            httpRequest.setAttribute("username", username);
            chain.doFilter(request, response);
        } else {
            httpResponse.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            httpResponse.getWriter().write("Unauthorized");
        }
    }
    
    private String extractToken(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        if (bearerToken != null && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }
}
```

## Filter Registration

### Using @Component with @Order

```java
@Component
@Order(1)
public class CustomFilter implements Filter {
    // Implementation
}
```

### Using FilterRegistrationBean

```java
@Configuration
public class FilterConfig {
    
    @Bean
    public FilterRegistrationBean<CustomFilter> loggingFilter() {
        FilterRegistrationBean<CustomFilter> registrationBean = new FilterRegistrationBean<>();
        
        registrationBean.setFilter(new CustomFilter());
        registrationBean.addUrlPatterns("/api/*");
        registrationBean.setOrder(1);
        
        return registrationBean;
    }
    
    @Bean
    public FilterRegistrationBean<CorsFilter> corsFilter() {
        FilterRegistrationBean<CorsFilter> registrationBean = new FilterRegistrationBean<>();
        
        registrationBean.setFilter(new CorsFilter());
        registrationBean.addUrlPatterns("/*");
        registrationBean.setOrder(Ordered.HIGHEST_PRECEDENCE);
        
        return registrationBean;
    }
}
```

## Advanced Patterns

### Request Tracing Filter

```java
@Component
@Order(1)
public class RequestTracingFilter implements Filter {
    
    private static final String TRACE_ID_HEADER = "X-Trace-ID";
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) 
            throws IOException, ServletException {
        
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;
        
        String traceId = httpRequest.getHeader(TRACE_ID_HEADER);
        if (traceId == null) {
            traceId = UUID.randomUUID().toString();
        }
        
        MDC.put("traceId", traceId);
        httpResponse.setHeader(TRACE_ID_HEADER, traceId);
        
        try {
            chain.doFilter(request, response);
        } finally {
            MDC.clear();
        }
    }
}
```

### Request Sanitization Filter

```java
@Component
@Order(2)
public class XssProtectionFilter implements Filter {
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) 
            throws IOException, ServletException {
        
        XssRequestWrapper wrappedRequest = new XssRequestWrapper((HttpServletRequest) request);
        chain.doFilter(wrappedRequest, response);
    }
    
    private static class XssRequestWrapper extends HttpServletRequestWrapper {
        
        public XssRequestWrapper(HttpServletRequest request) {
            super(request);
        }
        
        @Override
        public String getParameter(String name) {
            String value = super.getParameter(name);
            return sanitize(value);
        }
        
        @Override
        public String[] getParameterValues(String name) {
            String[] values = super.getParameterValues(name);
            if (values == null) return null;
            
            String[] sanitizedValues = new String[values.length];
            for (int i = 0; i < values.length; i++) {
                sanitizedValues[i] = sanitize(values[i]);
            }
            return sanitizedValues;
        }
        
        private String sanitize(String value) {
            if (value == null) return null;
            return value.replaceAll("<", "&lt;")
                       .replaceAll(">", "&gt;")
                       .replaceAll("\"", "&quot;")
                       .replaceAll("'", "&#x27;");
        }
    }
}
```

### Compression Filter

```java
@Component
@Order(4)
public class GzipCompressionFilter implements Filter {
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) 
            throws IOException, ServletException {
        
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;
        
        String acceptEncoding = httpRequest.getHeader("Accept-Encoding");
        
        if (acceptEncoding != null && acceptEncoding.contains("gzip")) {
            GzipResponseWrapper wrappedResponse = new GzipResponseWrapper(httpResponse);
            chain.doFilter(request, wrappedResponse);
            wrappedResponse.finish();
        } else {
            chain.doFilter(request, response);
        }
    }
}
```

## Filter Chain Execution Order

```
Request
  ↓
Filter 1 (Order: HIGHEST_PRECEDENCE)
  ↓
Filter 2 (Order: 1)
  ↓
Filter 3 (Order: 2)
  ↓
Spring Security Filter Chain
  ↓
DispatcherServlet
  ↓
Interceptors
  ↓
Controller
  ↓
Response (reverse order through filters)
```

## Best Practices

1. **Order filters appropriately** - Security and CORS filters should be first
2. **Always call chain.doFilter()** - Unless you want to block the request
3. **Clean up resources** - Use try-finally blocks
4. **Use wrapper classes** - For reading request/response multiple times
5. **Avoid heavy processing** - Filters execute on every request
6. **Handle exceptions** - Properly catch and handle ServletException and IOException

## Testing

```java
@SpringBootTest
@AutoConfigureMockMvc
class FilterTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Test
    void testCorsFilter() throws Exception {
        mockMvc.perform(options("/api/test")
                .header("Origin", "http://localhost:3000"))
                .andExpect(status().isOk())
                .andExpect(header().exists("Access-Control-Allow-Origin"));
    }
    
    @Test
    void testAuthenticationFilter_WithValidToken() throws Exception {
        mockMvc.perform(get("/api/protected")
                .header("Authorization", "Bearer valid-token"))
                .andExpect(status().isOk());
    }
    
    @Test
    void testAuthenticationFilter_WithoutToken() throws Exception {
        mockMvc.perform(get("/api/protected"))
                .andExpect(status().isUnauthorized());
    }
    
    @Test
    void testRequestTracingFilter() throws Exception {
        mockMvc.perform(get("/api/test"))
                .andExpect(header().exists("X-Trace-ID"));
    }
}
```

## Common Pitfalls

1. Not calling chain.doFilter() - Request stops processing
2. Reading request body multiple times without wrapper
3. Incorrect filter ordering
4. Not handling exceptions in filters
5. Blocking operations in filters

## Resources

- [Spring Boot Filter Documentation](https://docs.spring.io/spring-boot/docs/current/reference/html/web.html#web.servlet.embedded-container.filters)
- [Java Servlet Filter Specification](https://jakarta.ee/specifications/servlet/5.0/jakarta-servlet-spec-5.0.html)