# Apache Tomcat — The Omnivore Knowledge Vault

> **Version Focus:** Tomcat 10.1.x (Jakarta Servlet 6.0, JSP 3.1, EL 5.0, WebSocket 2.1)
> Prior versions noted where relevant.

---

## 🔹 Section 0: The "Elevator Pitch" & Core Philosophy

Apache Tomcat is an **open-source implementation of the Jakarta Servlet, JSP, EL, and WebSocket specifications** — it's a *servlet container* (web container), not a full Jakarta EE application server. It executes Java servlets and renders Java Server Pages.

**The Golden Rule:** Tomcat is a **Pipeline of Valves** around a **hierarchy of Containers** (`Engine → Host → Context → Wrapper`). Every request flows through a chain of responsibility — a Connector accepts it, a Pipeline processes it through Valves, and the terminal BasicValve hands it to the target servlet via the Wrapper.

**Why it exists:** To provide a standardized, portable runtime for Java web applications per the Servlet specification. It's lightweight compared to full EE servers (GlassFish, WildFly), embeddable, and the most widely deployed servlet container.

**What it is NOT:** Tomcat is NOT a full Jakarta EE application server — no EJB container, no JTA, no JMS (though it can connect to them). It IS a production-grade HTTP server + servlet/JSP engine.

💡 **Version migration matters more than most projects:** Tomcat 10+ migrated from `javax.*` to `jakarta.*` namespaces. Tomcat 9 is the last `javax.*` version. **Never mix them.**

---

## 🔹 Section 1: Foundational Anatomy (The "Big Picture" Map)

### High-Level Architecture Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                        SERVER                                │
│  (org.apache.catalina.core.StandardServer)                   │
│  ├── GlobalNamingResources (JNDI)                            │
│  └── Service ("Catalina")                                    │
│      ├── Connector (HTTP/1.1 NIO)           port 8080        │
│      ├── Connector (AJP/1.3 NIO)            port 8009        │
│      ├── Connector (HTTP/2)                 port 8443        │
│      └── Engine (Pipeline + BasicValve)                      │
│          ├── Host (localhost)                                │
│          │   ├── Context (/myapp)                            │
│          │   │   ├── Wrapper (MyServlet)                     │
│          │   │   ├── Wrapper (OtherServlet)                  │
│          │   │   └── JSP (compiled to Wrapper)               │
│          │   ├── Context (/ROOT)                             │
│          │   │   └── Wrapper (default)                       │
│          │   └── HostConfig (auto-deploy listener)           │
│          └── Realm (auth backend)                            │
│              └── UserDatabase (tomcat-users.xml)             │
└──────────────────────────────────────────────────────────────┘
```

### Core Components

| Component | Interface | Responsibility |
|---|---|---|
| **Server** | `org.apache.catalina.Server` | Top-level singleton; owns Services; manages JVM lifecycle |
| **Service** | `org.apache.catalina.Service` | Groups Connectors with exactly one Engine |
| **Connector** | `org.apache.catalina.Connector` | Listens on TCP port; parses HTTP/AJP bytes; creates Request/Response objects |
| **Engine** | `org.apache.catalina.Engine` | Request processing pipeline for a Service; routes to Host by hostname |
| **Host** | `org.apache.catalina.Host` | Virtual host; deploys/undeploys Contexts; owns appBase |
| **Context** | `org.apache.catalina.Context` | Single web application; contains Wrappers, session manager, classloader |
| **Wrapper** | `org.apache.catalina.Wrapper` | Single servlet; manages init/service/destroy lifecycle |
| **Pipeline** | `org.apache.catalina.Pipeline` | Ordered chain of Valves + one terminal BasicValve |
| **Valve** | `org.apache.catalina.Valve` | Single processing unit in the chain of responsibility |
| **Realm** | `org.apache.catalina.Realm` | Authentication/authorization data source |
| **Loader** | `org.apache.catalina.Loader` | Webapp ClassLoader for a Context |
| **Manager** | `org.apache.catalina.Manager` | Session creation, tracking, persistence, expiry |

### Request Lifecycle (Step-by-Step Flow)

```
 1. TCP connection arrives on Connector port
 2. Acceptor thread accepts socket
 3. Poller thread (NIO) detects readable event
 4. Processor parses HTTP request line + headers
 5. CoyoteAdapter.service() adapts Coyote Request → Catalina Request
 6. Engine.getPipeline().getFirst().invoke()
    └── Custom Engine-level Valves (authentication, access logging)
    └── StandardEngineValve → selects Host by serverName
       └── Host Valves (access control, rewrite)
       └── StandardHostValve → selects Context by path
          └── Context Valves (rate limiting, session persistence)
          └── StandardContextValve → selects Wrapper by URL pattern
             └── Wrapper Valves
             └── StandardWrapperValve → creates FilterChain
                └── Application Filters
                └── servlet.service(request, response)
 7. Response bytes flow back through Connector socket
 8. Post-processing: recycle request/response objects
```

---

## 🔹 Section 2: The Lexicon (Glossary of Terms)

### Core Jargon

| Term | Aliases | Atomic Definition | Context/Scope |
|---|---|---|---|
| **Catalina** | — | Tomcat's servlet container implementation (the Engine + containers) | Core — `org.apache.catalina.*` |
| **Coyote** | — | Tomcat's HTTP/AJP connector framework (network I/O) | `org.apache.coyote.*` |
| **Jasper** | JSP Engine | Compiles JSP files into servlets, handles EL expression evaluation | Inside each Context |
| **Valve** | — | Interceptor in the request processing pipeline — chain-of-responsibility | Per Container (Engine/Host/Context/Wrapper) |
| **Pipeline** | — | Ordered collection of Valves + exactly 1 terminal BasicValve | One per Container |
| **BasicValve** | Container Valve | The final Valve that invokes the next Container's pipeline | Engine→Host→Context→Wrapper chain |
| **Wrapper** | Servlet Wrapper | Manages servlet instance, init params, load-on-startup | Under Context (1 per servlet) |
| **Realm** | Auth Realm | User/role/password store for container-managed security | Engine/Host/Context |
| **Webapp** | Application, WAR | A Context's deployed application (WAR file or exploded directory) | Under Host |
| **Digester** | — | Apache Commons Digester — XML → Java object mapper (server.xml parsing) | Startup only |
| **Lifecycle** | — | State machine: NEW → INIT → START → STOP → DESTROY | All major components |
| **WebappClassLoader** | Webapp loader | Per-webapp classloader (parent-last for app classes, parent-first for EE API) | Each Context |
| **JNDI** | — | Java Naming and Directory Interface — resource lookup for DataSources, etc. | Global + per-webapp |
| **AJP** | AJP/1.3 | Apache JServ Protocol — binary wire protocol between Apache HTTPD and Tomcat | Connector type, port 8009 |

### Directory & File Terms

| Entity | Default Location | Purpose |
|---|---|---|
| `CATALINA_HOME` | `/opt/tomcat/` | Tomcat installation root (binaries, lib, shared) |
| `CATALINA_BASE` | Same as HOME | Instance-specific config; allows multiple instances from one HOME |
| `server.xml` | `$CATALINA_BASE/conf/` | Master configuration file |
| `web.xml` | `$CATALINA_BASE/conf/` | Global (default) web.xml — applies to all webapps |
| `tomcat-users.xml` | `$CATALINA_BASE/conf/` | User + role definitions for Realm |
| `catalina.properties` | `$CATALINA_BASE/conf/` | Server/shared loader paths, security settings |
| `catalina.policy` | `$CATALINA_BASE/conf/` | Java Security Manager policy (removed in Tomcat 11) |
| `catalina.sh` | `$CATALINA_HOME/bin/` | Startup/shutdown script (Unix) |
| `catalina.bat` | `$CATALINA_HOME/bin/` | Startup/shutdown script (Windows) |
| `appBase` | `webapps/` | Host-level directory for auto-deployed webapps |
| `docBase` | (per Context) | Webapp content root (pointed to by Context) |
| `work/` | `$CATALINA_BASE/work/` | Compiled JSP, serialized sessions (scratch space) |

### Web Application Deployment Descriptors

| File | Location | Purpose |
|---|---|---|
| `web.xml` | `WEB-INF/web.xml` | Servlet, filter, listener, security config |
| `context.xml` | `META-INF/context.xml` | Tomcat-specific context config (JNDI, session, resources) |
| `web-fragment.xml` | `META-INF/web-fragment.xml` | Modular web fragment (Servlet 3.0+) |
| `taglib.tld` | `WEB-INF/` or `META-INF/` | Tag library descriptor |
| `MANIFEST.MF` | `META-INF/MANIFEST.MF` | JAR metadata (Class-Path, extensions) |

---

## 🔹 Section 3: Deep Dive — Memory & Resource Architecture

### ClassLoader Hierarchy

```
┌────────────────────────────────────────────────────────────────┐
│  Bootstrap ClassLoader                                         │
│  JVM core (rt.jar/java.base) — NOT accessible from webapps    │
└────────────────────────────────────────────────────────────────┘
        ↑ delegates to parent
┌────────────────────────────────────────────────────────────────┐
│  System ClassLoader                                            │
│  CLASSPATH, $CATALINA_HOME/bin/ (bootstrap.jar, tomcat-juli)  │
└────────────────────────────────────────────────────────────────┘
        ↑
┌────────────────────────────────────────────────────────────────┐
│  Common ClassLoader                                            │
│  $CATALINA_HOME/lib/, $CATALINA_BASE/lib/                      │
│  (servlet-api.jar, catalina.jar, jasper.jar, etc.)             │
└────────────────────────────────────────────────────────────────┘
        ↑              ↑                              ↑
  (server.loader)  (shared.loader)        (per-webapp ClassLoaders)
        ↓              ↓                              ↓
┌──────────────┐ ┌──────────────┐  ┌──────────────────────────────┐
│ Server CL    │ │ Shared CL    │  │ WebappClassLoader [/myapp]   │
│ Tomcat       │ │ Visible to   │  │ Loads from:                   │
│ internals    │ │ ALL webapps  │  │ 1. /WEB-INF/classes           │
│ only         │ │              │  │ 2. /WEB-INF/lib/*.jar         │
└──────────────┘ └──────────────┘  │ Delegation: PARENT-FIRST     │
                                    │ for EE APIs (Servlet, JSP)   │
                                    │ PARENT-LAST for everything   │
                                    └──────────────────────────────┘
```

**⚠️ Webapp ClassLoader Delegation Rule (Servlet Spec 6.0 §9.7.2):**

| Class Type | Delegation Order |
|---|---|
| JRE classes (`java.*`, `javax.*` / `jakarta.*` EE APIs) | Parent-first (cannot override) |
| XML parser components (upgradeable modules) | Parent-first (can override via upgradeable modules) |
| Everything else (app classes, frameworks, drivers) | **Look in webapp FIRST**, delegate to parent only if not found |

This means a library in `WEB-INF/lib` **overrides** the same library in `$CATALINA_HOME/lib` — except for the Jakarta EE API classes.

### Memory Regions

```
JVM Process Memory
├── Heap
│   ├── Young Generation (Eden + S0 + S1)    ← Short-lived (request objects, etc.)
│   └── Old Generation (Tenured)             ← Long-lived (sessions, caches, Spring beans)
├── Metaspace (Java 8+)                      ← Class metadata, JIT data, loaded classes
│   ├── Class metadata (method area)
│   ├── Constant pool per class
│   ├── JIT compiled code (code cache)
│   └── WebappClassLoader instances           ← *** #1 LEAK SOURCE ***
├── Thread Stacks                            ← ~1MB per thread (configurable with -Xss)
├── Direct Memory (ByteBuffer.allocateDirect) ← NIO buffers
└── Native Memory (JNI, zlib, APR)           ← ZipFile handles, SSL contexts
```

### ClassLoader Leak Mechanics — THE #1 TOMCAT PRODUCTION BUG

**The leak chain (why one leak pins EVERYTHING):**

```
External registry (DriverManager, Thread, JMX, Logger)
  → holds reference → Class<?> loaded by WebappClassLoader
    → holds reference → WebappClassLoader instance
      → holds reference → ALL classes it loaded (potentially thousands)
        → holds reference → ALL static fields → ALL referenced objects
```

After reloading a webapp, you now have **two** copies of every class in memory. Reload enough times → `OutOfMemoryError: Metaspace`.

| Source | Mechanism | Tomcat Mitigation | You Must Also |
|---|---|---|---|
| `ThreadLocal` not `.remove()`'d | Thread pool thread holds TCCL ref via value's class | Thread renewal (Tomcat 7+) | Call `ThreadLocal.remove()` in `contextDestroyed` |
| JDBC driver in `WEB-INF/lib` | `DriverManager` (system CL) holds ref | Auto-deregistration on stop | Better: put JDBC driver in `$CATALINA_HOME/lib` |
| `java.util.logging` Handler | Logger refs Handler → Handler class → webapp CL | `JreMemoryLeakPreventionListener` | Properly close loggers |
| `java.beans.Introspector` | Static `BeanInfo` cache | Flushed on context stop | Use `Introspector.flushFromCaches()` or Apache BeanUtils |
| Timer threads not stopped | Thread's context ClassLoader = webapp CL | Logs error on stop | `timer.cancel()` in `contextDestroyed` |
| ExecutorService not shut down | Alive threads hold TCCL | None | `executor.shutdown()` in `contextDestroyed` |
| RMI/JMX registrations | Registry (system CL) holds ref to stub | MBean cleanup on shutdown | `unexportObject()` / `unregisterMBean()` |
| Quartz Scheduler | Scheduler threads hold TCCL | None | `scheduler.shutdown()` in `contextDestroyed` |
| ThreadPoolExecutor + `ThreadLocal` | Thread reused across redeploy | Thread renewal helps | Clean ThreadLocals, stop executor |

💡 **Detection on reload:** After `reloadable="true"` triggers a reload, check the Manager app "Find Leaks" button or use a profiler. Look for `org.apache.catalina.loader.WebappClassLoaderBase` where `started = false` — those are leaked.

💡 **JVM flags to help:** `-XX:+TraceClassLoading` / `-XX:+TraceClassUnloading` — watch class loading/unloading during redeploy.

### Threading Model

```
===========================================================
  Connector Thread Pool (NIO)
===========================================================
  ├─ Acceptor Threads (1-2)      ← accepts TCP connections
  │                               ← registers sockets with NIO Selector
  ├─ Poller Threads (1-2)        ← polls Selector for readable events
  │                               ← creates Processor, hands to Worker
  └─ Worker Thread Pool          ← processes HTTP + servlet
      maxThreads (default 200)    ← each request gets one thread
      minSpareThreads (default 10)← kept alive waiting for work
===========================================================
  Container Background Thread
===========================================================
  backgroundProcessorDelay=10s   ← session expiry, webapp reload
===========================================================
  Async Servlet Threads
===========================================================
  request.startAsync()           ← releases worker thread
  asyncContext.start(() -> ...)  ← continues in separate thread
```

**Thread pool sizing:**
```
Optimal maxThreads ≈ (cores * 2) / (1 - blocking_coefficient)

Blocking coefficient:
  - CPU-bound (no I/O wait):       0.0  → maxThreads ≈ cores
  - Light I/O (cache-hit DB):      0.2  → maxThreads ≈ cores * 2.5
  - Typical (DB, REST calls):      0.5  → maxThreads ≈ cores * 4
  - Heavy I/O (file serving):      0.8  → maxThreads ≈ cores * 10
```

### I/O Models Comparison

| Connector | I/O | Thread/Conn | Best For | Deprecated? |
|---|---|---|---|---|
| `HTTP/1.1` NIO (default) | Non-blocking NIO channels | ~1:1 active, pooled | General purpose | No |
| `HTTP/1.1` NIO2 | Async NIO channels | Fewer threads | High connection count | No |
| `HTTP/1.1` APR | Native epoll/kqueue via JNI | Native perf | Legacy deployments | **Yes** (10.1+) |
| `AJP/1.3` NIO | Binary protocol | Same as NIO | Behind Apache HTTPD | No (but AJP deprecated in general) |

---

## 🔹 Section 4: The Commandments (Configuration & Core Operations)

### `server.xml` — Master Configuration

**Top-level skeleton:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Server port="8005" shutdown="SHUTDOWN">
  <Listener className="org.apache.catalina.startup.VersionLoggerListener" />
  <Listener className="org.apache.catalina.core.AprLifecycleListener" />
  <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
  <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
  <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />

  <GlobalNamingResources>
    <Resource name="UserDatabase" auth="Container"
              type="org.apache.catalina.UserDatabase"
              description="User database"
              factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
              pathname="conf/tomcat-users.xml" />
  </GlobalNamingResources>

  <Service name="Catalina">
    <Executor name="tomcatThreadPool" namePrefix="catalina-exec-"
              maxThreads="200" minSpareThreads="10" />

    <Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443"
               executor="tomcatThreadPool"
               maxKeepAliveRequests="100"
               compression="off" />

    <Connector port="8009" protocol="AJP/1.3"
               redirectPort="8443" secretRequired="true" />

    <Engine name="Catalina" defaultHost="localhost" jvmRoute="worker1">
      <Realm className="org.apache.catalina.realm.LockOutRealm">
        <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
               resourceName="UserDatabase" />
      </Realm>

      <Host name="localhost" appBase="webapps"
            unpackWARs="true" autoDeploy="true">
        <Valve className="org.apache.catalina.valves.AccessLogValve"
               directory="logs" prefix="localhost_access_log"
               suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
        
        <Context path="" docBase="ROOT" />
      </Host>
    </Engine>
  </Service>
</Server>
```

### Connector Configuration — Critical Attributes

| Attribute | Default | Production Recommendation | Why |
|---|---|---|---|
| `maxThreads` | 200 | 200-800 (depends on hardware) | Controls concurrency ceiling |
| `minSpareThreads` | 10 | 25-50 | Prevents burst thread-creation overhead |
| `acceptCount` | 100 | 300-500 | Backlog queue when all threads busy |
| `maxConnections` | 8192 (NIO) | 10000 | Max simultaneous connections before OS-level reject |
| `connectionTimeout` | 60000 | 20000-30000 (ms) | Time to wait for request line |
| `keepAliveTimeout` | derived from connectionTimeout | 5000-15000 | Max time between requests on Keep-Alive |
| `maxKeepAliveRequests` | 100 | 100-1000 | Max pipelined requests before closing |
| `enableLookups` | false | **false** | Reverse DNS = expensive, slow |
| `compression` | off | `on` or `force` (if frontend doesn't handle) | Gzip response bodies |
| `compressibleMimeType` | text/html,... | add `application/json` | Extend compression to API responses |
| `maxHttpHeaderSize` | 8192 (8K) | 32768 (32K) | Larger headers for SSO tokens, JWTs |
| `maxPostSize` | 2097152 (2MB) | Adjust per need | POST body size limit |
| `URIEncoding` | ISO-8859-1 | `UTF-8` | **CRITICAL**: Fixes URL encoding issues |

### Container Hierarchy Configuration

**Engine-level:**
```xml
<Engine name="Catalina" defaultHost="localhost" jvmRoute="worker1"
        backgroundProcessorDelay="10"
        startStopThreads="0">  <!-- 0 = use Runtime.availableProcessors() -->
```

| Attribute | Default | Description |
|---|---|---|
| `jvmRoute` | (none) | Unique node ID for sticky sessions with load balancer |
| `backgroundProcessorDelay` | 10 | Seconds between background processing calls |
| `startStopThreads` | 1 | Threads for parallel Host startup (0=availableProcessors) |

**Host-level:**
```xml
<Host name="www.example.com" appBase="webapps"
      unpackWARs="true" autoDeploy="true"
      deployOnStartup="true">
```

| Attribute | Default | Description |
|---|---|---|
| `appBase` | `webapps` | Directory for auto-deployed apps |
| `autoDeploy` | true | Deploy new/updated apps while running |
| `deployOnStartup` | true | Deploy apps in appBase at startup |
| `unpackWARs` | true | Expand WAR files into directories |
| `failCtxIfServletStartFails` | false | Fail context if any servlet fails to start |

**Context-level:**
```xml
<Context path="/myapp" docBase="/opt/apps/myapp"
         reloadable="false"    <!-- false in production -->
         crossContext="false"  <!-- prevent app A reading app B's sessions -->
         privileged="false"
         sessionTimeout="30"
         cookies="true"
         useHttpOnly="true">
```

| Attribute | Default | Production | Description |
|---|---|---|---|
| `reloadable` | false | **false** | Watch WEB-INF/classes + WEB-INF/lib for changes — hot reload |
| `crossContext` | false | **false** | Allow `ServletContext.getContext()` across apps |
| `privileged` | false | **false** | Allow access to Tomcat internal servlets |
| `sessionTimeout` | 30 | 15-30 | Minutes of inactivity before session expires |
| `swallowAbortedUploads` | true | true | Prevent `IOException` from broken multipart uploads |

### `web.xml` Essential Elements

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="https://jakarta.ee/xml/ns/jakartaee"
         version="6.0"
         metadata-complete="false">
  
  <!-- Context Params -->
  <context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>/WEB-INF/applicationContext.xml</param-value>
  </context-param>
  
  <!-- Listeners -->
  <listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
  </listener>
  
  <!-- Filters -->
  <filter>
    <filter-name>encodingFilter</filter-name>
    <filter-class>org.apache.catalina.filters.SetCharacterEncodingFilter</filter-class>
    <init-param>
      <param-name>encoding</param-name>
      <param-value>UTF-8</param-value>
    </init-param>
  </filter>
  <filter-mapping>
    <filter-name>encodingFilter</filter-name>
    <url-pattern>/*</url-pattern>
  </filter-mapping>
  
  <!-- Servlets -->
  <servlet>
    <servlet-name>dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <load-on-startup>1</load-on-startup>
    <multipart-config>
      <max-file-size>10485760</max-file-size>
      <max-request-size>20971520</max-request-size>
      <file-size-threshold>5242880</file-size-threshold>
    </multipart-config>
  </servlet>
  <servlet-mapping>
    <servlet-name>dispatcher</servlet-name>
    <url-pattern>/</url-pattern>
  </servlet-mapping>
  
  <!-- Session -->
  <session-config>
    <session-timeout>30</session-timeout>
    <cookie-config>
      <http-only>true</http-only>
      <secure>true</secure>
      <same-site>Lax</same-site>
    </cookie-config>
    <tracking-mode>COOKIE</tracking-mode>
  </session-config>
  
  <!-- Error Pages -->
  <error-page>
    <error-code>404</error-code>
    <location>/WEB-INF/views/error/404.jsp</location>
  </error-page>
  <error-page>
    <exception-type>java.lang.Exception</exception-type>
    <location>/WEB-INF/views/error/general.jsp</location>
  </error-page>
  
  <!-- MIME type mappings -->
  <mime-mapping>
    <extension>wasm</extension>
    <mime-type>application/wasm</mime-type>
  </mime-mapping>
</web-app>
```

### Lifecycle — Starting & Stopping

**Startup sequence:**
```
1. JVM starts → runs catalina.sh
2. Bootstrap.main() → creates Catalina instance
3. Catalina.load() → 
   a. Initialize classloaders (common, server, shared)
   b. Digester parses server.xml → creates Server, Services, Connectors, Engine, Host, Contexts
   c. Calls Server.init() → Service.init() → Connector.init() → Engine.init()
4. Catalina.start() →
   a. Server.start() → fires lifecycle events
   b. HostConfig fires → deploys webapps from appBase
   c. ContextConfig fires → parses web.xml + annotations
   d. For each Context: Loader.start, Manager.start, Wrapper.start (servlet.init())
5. Server.await() → main thread blocks, waits for SHUTDOWN command on port 8005
```

**Shutdown sequence:**
```
1. SHUTDOWN command received (or SIGTERM/Ctrl+C)
2. Server.stop() → Service.stop() → Connector.pause() → Engine.stop()
3. Each Context: Wrapper.stop() → servlet.destroy()
4. Manager.stop() → session persistence if configured
5. HostConfig.stop(), Loader.stop()
6. Connector.stop() → unbind ports
7. Server.destroy() → cleanup
8. JVM exits
```

### Custom Valve Implementation

```java
package com.example;

import org.apache.catalina.valves.ValveBase;
import org.apache.catalina.connector.Request;
import org.apache.catalina.connector.Response;
import javax.servlet.ServletException;
import java.io.IOException;

public class RequestTimingValve extends ValveBase {

    private String name = "TimingValve";

    @Override
    public void invoke(Request request, Response response)
            throws IOException, ServletException {

        long start = System.nanoTime();
        try {
            getNext().invoke(request, response);  // ⚡ pass to next in pipeline
        } finally {
            long elapsed = System.nanoTime() - start;
            log.info("[{}] {} {} => {} ms",
                name,
                request.getMethod(),
                request.getRequestURI(),
                elapsed / 1_000_000);
        }
    }

    public void setName(String name) { this.name = name; }
}
```

**Deployment — place JAR in `$CATALINA_HOME/lib/` and configure:**

```xml
<Host name="localhost" appBase="webapps">
  <Valve className="com.example.RequestTimingValve"
         name="MyTimer" />
</Host>
```

---

## 🔹 Section 5: The Standard Library / Core API Blueprint

### Built-in Valve Types

| Valve Class | Function | Configure On |
|---|---|---|
| `AccessLogValve` | Writes NCSA combined/common access log | Engine/Host/Context |
| `RemoteAddrValve` | Allow/deny by client IP regex | Engine/Host/Context |
| `RemoteHostValve` | Allow/deny by client hostname regex | Engine/Host/Context |
| `RemoteCIDRValve` | Allow/deny by CIDR notation | Engine/Host/Context |
| `RewriteValve` | URL rewriting (mod_rewrite-like) | Host |
| `ErrorReportValve` | Customizes error pages; show/hide server info | Engine/Host/Context |
| `StuckThreadDetectionValve` | Detects threads stuck > threshold; logs, optionally kills | Engine |
| `LoadBalancerDrainingValve` | Drains sessions from disabled load-balancer nodes | Engine/Host/Context |
| `PersistentValve` | Store sessions before request, restore after | Context |
| `SSLValve` | Extracts SSL info from proxy headers | Engine/Host/Context |
| `CrawlerSessionManagerValve` | Prevents crawler-induced session churn | Context |
| `RequestFilterValve` | Base for IP/hostname access control | Engine/Host/Context |
| `HealthCheckValve` | Exposes Tomcat health via HTTP | Host/Context |
| `AuthenticatorValves` (`Basic`, `Digest`, `Form`, `SSL`, `SPNEGO`) | Container-managed authentication | Context |
| `NonLoginAuthenticatorValve` | Processes `CONFIDENTIAL`/`INTEGRAL` without login | Context |
| `SSIAccessValve` | Server-Side Includes access control | Context |
| `JsonErrorReportValve` | Error responses in JSON format (10.1+) | Engine/Host/Context |

### Common Valve Configurations

**Access Log (NCSA Combined):**
```xml
<Valve className="org.apache.catalina.valves.AccessLogValve"
       directory="logs"
       prefix="access" suffix=".log"
       pattern="%h %l %u %t &quot;%r&quot; %s %b %{Referer}i %{User-Agent}i"
       rotatable="true" fileDateFormat="yyyy-MM-dd" />
```
Pattern elements: `%h`=remote host, `%l`=identd, `%u`=auth user, `%t`=timestamp, `%r`=request line, `%s`=status, `%b`=bytes, `%D`=time(ms), `%T`=time(sec), `%I`=current thread name, `%F`=time taken to commit response

**Remote IP Restriction (CIDR):**
```xml
<Valve className="org.apache.catalina.valves.RemoteCIDRValve"
       allow="127\.0\.0\.1|::1|10\.0\.\d+\.\d+" />
```

**Stuck Thread Detection:**
```xml
<Valve className="org.apache.catalina.valves.StuckThreadDetectionValve"
       threshold="60" />  <!-- seconds — log WARN for threads stuck > 60s -->
```

**Rewrite (mod_rewrite style) — in `$CATALINA_BASE/conf/Catalina/localhost/rewrite.config`:**
```
RewriteRule ^/api/(.*)$ /api-dispatcher/$1 [L]
RewriteCond %{REQUEST_URI} !^/assets/
RewriteRule ^/(.*)$ /app/$1 [L]
```

### Built-in Container-Provided Filters

**CORS Filter:**
```xml
<filter>
  <filter-name>CorsFilter</filter-name>
  <filter-class>org.apache.catalina.filters.CorsFilter</filter-class>
  <init-param>
    <param-name>cors.allowed.origins</param-name>
    <param-value>https://myapp.example.com</param-value>
  </init-param>
  <init-param>
    <param-name>cors.allowed.methods</param-name>
    <param-value>GET,POST,PUT,DELETE,OPTIONS</param-value>
  </init-param>
  <init-param>
    <param-name>cors.allowed.headers</param-name>
    <param-value>Content-Type,Authorization,X-Requested-With</param-value>
  </init-param>
  <init-param>
    <param-name>cors.support.credentials</param-name>
    <param-value>true</param-value>
  </init-param>
  <init-param>
    <param-name>cors.preflight.maxage</param-name>
    <param-value>3600</param-value>
  </init-param>
</filter>
<filter-mapping>
  <filter-name>CorsFilter</filter-name>
  <url-pattern>/api/*</url-pattern>
</filter-mapping>
```

**HTTP Header Security Filter:**
```xml
<filter>
  <filter-name>httpHeaderSecurity</filter-name>
  <filter-class>org.apache.catalina.filters.HttpHeaderSecurityFilter</filter-class>
  <init-param>
    <param-name>hstsEnabled</param-name>
    <param-value>true</param-value>
  </init-param>
  <init-param>
    <param-name>hstsMaxAgeSeconds</param-name>
    <param-value>31536000</param-value>
  </init-param>
  <init-param>
    <param-name>antiClickJackingEnabled</param-name>
    <param-value>true</param-value>
  </init-param>
  <init-param>
    <param-name>antiClickJackingOption</param-name>
    <param-value>SAMEORIGIN</param-value>
  </init-param>
  <init-param>
    <param-name>blockContentTypeSniffingEnabled</param-name>
    <param-value>true</param-value>
  </init-param>
</filter>
<filter-mapping>
  <filter-name>httpHeaderSecurity</filter-name>
  <url-pattern>/*</url-pattern>
</filter-mapping>
```

**CSRF Prevention Filter:**
```xml
<filter>
  <filter-name>csrfFilter</filter-name>
  <filter-class>org.apache.catalina.filters.CsrfPreventionFilter</filter-class>
  <init-param>
    <param-name>entryPoints</param-name>
    <param-value>/login,/login.jsp</param-value>
  </init-param>
</filter>
<filter-mapping>
  <filter-name>csrfFilter</filter-name>
  <url-pattern>/manager/html/*</url-pattern>
</filter-mapping>
```

**Set Character Encoding:**
```xml
<filter>
  <filter-name>encodingFilter</filter-name>
  <filter-class>org.apache.catalina.filters.SetCharacterEncodingFilter</filter-class>
  <init-param>
    <param-name>encoding</param-name>
    <param-value>UTF-8</param-value>
  </init-param>
  <init-param>
    <param-name>ignore</param-name>
    <param-value>false</param-value>
  </init-param>
</filter>
<filter-mapping>
  <filter-name>encodingFilter</filter-name>
  <url-pattern>/*</url-pattern>
</filter-mapping>
```

### JDBC DataSource Configuration

**In `context.xml` or `server.xml` GlobalNamingResources:**

```xml
<Resource name="jdbc/MyDB" auth="Container"
          type="javax.sql.DataSource"
          factory="org.apache.tomcat.jdbc.pool.DataSourceFactory"
          driverClassName="org.postgresql.Driver"
          url="jdbc:postgresql://localhost:5432/mydb"
          username="app_user"
          password="${db.password}"    <!-- Use -D property substitution -->
          
          maxActive="100"
          maxIdle="30"
          minIdle="10"
          initialSize="10"
          maxWaitMillis="5000"
          
          testOnBorrow="true"
          testWhileIdle="true"
          validationQuery="SELECT 1"
          validationInterval="30000"
          timeBetweenEvictionRunsMillis="30000"
          minEvictableIdleTimeMillis="60000"
          
          removeAbandonedTimeout="60"
          removeAbandoned="true"
          logAbandoned="true"
          
          jdbcInterceptors="ConnectionState;StatementFinalizer"
          closeMethod="close" />
```

**In `web.xml`:**
```xml
<resource-ref>
  <res-ref-name>jdbc/MyDB</res-ref-name>
  <res-type>javax.sql.DataSource</res-type>
  <res-auth>Container</res-auth>
</resource-ref>
```

### Tomcat JDBC Pool — Key Interceptors

| Interceptor | Purpose |
|---|---|
| `ConnectionState` | Cache `setAutoCommit`, `setReadOnly`, `setCatalog`, `setTransactionIsolation` — avoids round-trips |
| `StatementFinalizer` | Automatically closes unclosed Statements when Connection returns to pool |
| `SlowQueryReport` | Log queries exceeding threshold; tracks statistics |
| `SlowQueryReportJmx` | Same + MBean registration for JMX monitoring |
| `ResetAbandonedTimer` | Reset abandoned timer when connection is used (avoids false positives on long transactions) |
| `Tracing` | Adds connection usage stack trace for debugging pool leaks |

---

## 🔹 Section 6: The Toolchain & Terminal Arsenal

### Installation & Version Managers

**Binary download:**
```bash
# Download
wget https://dlcdn.apache.org/tomcat/tomcat-10/v10.1.34/bin/apache-tomcat-10.1.34.tar.gz

# Extract
tar xzf apache-tomcat-10.1.34.tar.gz
mv apache-tomcat-10.1.34 /opt/tomcat/

# Create symlink
ln -s /opt/tomcat/apache-tomcat-10.1.34 /opt/tomcat/latest
```

**Homebrew (macOS):**
```bash
brew install tomcat@10
brew services start tomcat@10
```

**Docker:**
```bash
docker pull tomcat:10.1-jdk21
docker run -d -p 8080:8080 -p 8005:8005 \
  -v /my/app:/usr/local/tomcat/webapps/myapp \
  -v /my/tomcat/conf:/usr/local/tomcat/conf \
  --name my-tomcat tomcat:10.1-jdk21
```

### Environment Variables

| Variable | Purpose | Example |
|---|---|---|
| `CATALINA_HOME` | Installation root | `/opt/tomcat/latest` |
| `CATALINA_BASE` | Instance config root (defaults to HOME) | `/var/tomcat/instance1` |
| `CATALINA_TMPDIR` | Temp directory | `/var/tomcat/temp` |
| `CATALINA_PID` | PID file path | `/var/run/tomcat.pid` |
| `JAVA_HOME` | JDK installation | `/usr/lib/jvm/java-21-openjdk` |
| `JRE_HOME` | JRE installation (if no JDK) | `/usr/lib/jvm/jre-21` |
| `JAVA_OPTS` | JVM flags (appended to JAVA_OPTS) | `-Xmx2g -Xms512m` |
| `CATALINA_OPTS` | JVM flags (only for start/run, not stop) | `-Djava.security.egd=file:/dev/./urandom` |

### Startup Scripts

```bash
# Start in foreground (Ctrl+C to stop)
$CATALINA_HOME/bin/catalina.sh run

# Start as daemon
$CATALINA_HOME/bin/startup.sh

# Stop
$CATALINA_HOME/bin/shutdown.sh

# Debug mode (port 8000 by default)
$CATALINA_HOME/bin/catalina.sh jpda start

# With custom JVM options
CATALINA_OPTS="-Xmx4g -Xms1g -XX:+UseZGC" $CATALINA_HOME/bin/catalina.sh run
```

### Common JVM Flags for Tomcat

```bash
export CATALINA_OPTS="
  -Xms2g -Xmx4g                    # Heap size
  -XX:MetaspaceSize=256m           # Initial metaspace
  -XX:MaxMetaspaceSize=512m        # Max metaspace (prevent uncontrolled growth)
  -XX:+UseZGC                      # Z Garbage Collector (JDK 21+)
  -XX:ConcGCThreads=2              # Concurrent GC threads
  -Xlog:gc*:file=$CATALINA_BASE/logs/gc.log:time,uptime,level,tags  # GC logging
  
  -Djava.awt.headless=true         # No display needed
  -Djava.security.egd=file:/dev/./urandom  # Faster SecureRandom startup
  -Djava.net.preferIPv4Stack=true  # Prefer IPv4
  
  -Dorg.apache.catalina.connector.RECYCLE_FACADES=true  # Security: discard request facades
  -Dorg.apache.tomcat.util.buf.UDecoder.ALLOW_ENCODED_SLASH=true  # Allow encoded slashes
  
  -Dfile.encoding=UTF-8            # Default file encoding
"
```

### Manager App — Essential Commands

The Manager webapp (`/manager/html` — GUI, `/manager/text` — text) requires a user with `manager-gui` or `manager-script` role.

```bash
# Text-based manager — curl commands
curl -u admin:password "http://localhost:8080/manager/text/list"
# Output: /myapp:running:0:myapp
#         /:running:0:ROOT

# Deploy WAR from local path
curl -u admin:password --upload-file myapp.war \
  "http://localhost:8080/manager/text/deploy?path=/myapp&update=true"

# Deploy from URL
curl -u admin:password \
  "http://localhost:8080/manager/text/deploy?path=/myapp&war=file:/opt/deploy/myapp.war"

# Undeploy
curl -u admin:password \
  "http://localhost:8080/manager/text/undeploy?path=/myapp"

# Start/Stop
curl -u admin:password \
  "http://localhost:8080/manager/text/start?path=/myapp"
curl -u admin:password \
  "http://localhost:8080/manager/text/stop?path=/myapp"

# Reload
curl -u admin:password \
  "http://localhost:8080/manager/text/reload?path=/myapp"

# Find classloader leaks
curl -u admin:password \
  "http://localhost:8080/manager/text/findleaks?statusLine=true"

# View JVM metrics
curl -u admin:password \
  "http://localhost:8080/manager/text/serverinfo"
# Output: OK - Server info
#         Tomcat Version: Apache Tomcat/10.1.34
#         JVM Version: 21.0.4
#         JVM Vendor: Eclipse Adoptium
#         OS Name: Linux
#         OS Version: 6.1.0
#         OS Architecture: amd64
#         Available Processors: 8
#         Free Memory: 1024 MB
#         Total Memory: 2048 MB
#         Max Memory: 4096 MB
```

### Debugging Session

```bash
# Enable remote debugging (port 8000)
export CATALINA_OPTS="-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:8000"
$CATALINA_HOME/bin/catalina.sh run

# Then attach with
jdb -attach localhost:8000
# Or
# IntelliJ / VS Code → Remote JVM Debug → port 8000
```

### Logging — Logback / Log4j Integration

By default Tomcat uses **JULI** (its own `java.util.logging` fork). Replace with Logback:

```bash
# 1. Remove old logging libs
rm $CATALINA_HOME/lib/tomcat-juli.jar

# 2. Add logback + jansi
# 3. Create $CATALINA_BASE/conf/logback.xml
# 4. Set -Dlogback.configurationFile=conf/logback.xml
```

**Standard log files (under `$CATALINA_BASE/logs/`):**

| File | Content |
|---|---|
| `catalina.out` | Tomcat stdout/stderr (System.out, uncaught exceptions) |
| `catalina.YYYY-MM-DD.log` | Tomcat internal log messages |
| `localhost.YYYY-MM-DD.log` | Webapp-specific logs (formatted logger) |
| `localhost_access_log.YYYY-MM-DD.txt` | Access log (if AccessLogValve configured) |
| `manager.YYYY-MM-DD.log` | Manager app logs |
| `host-manager.YYYY-MM-DD.log` | Host Manager logs |

---

## 🔹 Section 7: Performance Tuning & Profiling

### Connector Tuning Checklist

```xml
<Connector port="8080" protocol="HTTP/1.1"
           maxThreads="400"
           minSpareThreads="50"
           acceptCount="500"
           maxConnections="10000"
           connectionTimeout="20000"
           keepAliveTimeout="10000"
           maxKeepAliveRequests="500"
           enableLookups="false"
           compression="on"
           compressibleMimeType="text/html,text/xml,text/plain,text/css,text/javascript,application/javascript,application/json"
           URIEncoding="UTF-8"
           maxSwallowSize="-1" />    <!-- -1 = swallow all aborted uploads -->
```

### Thread Pool Executor (for shared pool across connectors)

```xml
<Executor name="tomcatThreadPool" namePrefix="catalina-exec-"
          maxThreads="400"
          minSpareThreads="50"
          maxIdleTime="60000"
          threadPriority="5"
          daemon="true" />
```

### JVM Garbage Collector Selection

| GC | JVM Flags | Best For | Tomcat Impact |
|---|---|---|---|
| **G1 GC** (default) | `-XX:+UseG1GC` | Balanced throughput + latency | Good default; tuning needed for >4GB heaps |
| **ZGC** (JDK 21+) | `-XX:+UseZGC` | Sub-millisecond pause times | Excellent for high-throughput APIs |
| **Shenandoah** (JDK 21+) | `-XX:+UseShenandoahGC` | Low pause, concurrent compaction | Similar to ZGC, different tradeoffs |
| **Parallel GC** | `-XX:+UseParallelGC` | Max throughput, longer pauses | Batch processing, not web apps |
| **CMS** (removed JDK 14) | — | — | Ancient history; don't use |

### Profiling Flow

1. **Quick pulse check** — Manager app `/manager/text/serverinfo` (free/total/max memory)
2. **Thread dump** — `jstack <pid>` or `kill -3 <pid>` (signal to catalina.out)
3. **Heap dump** — `jmap -dump:live,format=b,file=heap.hprof <pid>`
4. **GC analysis** — `jstat -gcutil <pid> 1s` → watch YGC/YGCT/FGCT
5. **Deep dive** — Eclipse MAT / YourKit on the heap dump

**Thread dump analysis — key patterns:**
```
"http-nio-8080-exec-42" #84 daemon prio=5 tid=0x...
    java.lang.Thread.State: BLOCKED (on object monitor)    ← contention
    java.lang.Thread.State: WAITING (parking)              ← waiting in pool
    java.lang.Thread.State: RUNNABLE                       ← actively working
    java.lang.Thread.State: TIMED_WAITING (sleeping)       ← sleeping/timing
```

**Most threads in WAITING?** → Underutilized (too many threads or too few requests).
**Most threads in RUNNABLE?** → CPU-bound (check CPU %).
**Many threads BLOCKED?** → Lock contention (synchronized bottleneck).

### Performance Optimization Checklist

| Area | Quick Win | Advanced |
|---|---|---|
| Thread pool | Set minSpareThreads high enough to avoid burst creation | Profile to find optimal maxThreads |
| Connection pool | Set validationInterval > 0 to avoid per-borrow validation | Use `StatementFinalizer` interceptor |
| Compression | Enable `compression="on"` for text types | Offload to reverse proxy (nginx) |
| Session persistence | Disable if not needed: `Manager pathname=""` | Use external session store (Redis) |
| Static resources | Set `cachingAllowed="true"` on Resources | Move static files to CDN/nginx |
| JSP compilation | Set `development="false"` in web.xml | Precompile with JspC |
| DNS | `enableLookups="false"` | — |
| Keep-Alive | Set `maxKeepAliveRequests=100` | Too high = slow clients hog threads |
| Logging | Async appender (Logback `AsyncAppender`) | Reduce log level to WARN in production |
| NIO vs NIO2 | NIO is stable default | NIO2 for very high connection counts |
| Senor URL | Disable if not using: context `swallowAbortedUploads` | — |

### Tomcat JDBC Pool — Performance Tuning

```xml
<Resource name="jdbc/MyDB" auth="Container"
          type="javax.sql.DataSource"
          factory="org.apache.tomcat.jdbc.pool.DataSourceFactory"
          ...
          maxActive="100"           <!-- max concurrent connections -->
          maxIdle="30"              <!-- max idle in pool -->
          minIdle="10"              <!-- keep at least 10 idle -->
          initialSize="10"          <!-- pre-create 10 on startup -->
          maxWaitMillis="5000"      <!-- throw exception after 5s waiting -->
          
          testOnBorrow="false"      <!-- DO NOT test every borrow -->
          testWhileIdle="true"      <!-- DO test idle connections -->
          validationQuery="SELECT 1"
          validationInterval="30000" <!-- max frequency of validation checks -->
          timeBetweenEvictionRunsMillis="5000"
          minEvictableIdleTimeMillis="30000"
          
          removeAbandonedTimeout="60"
          removeAbandoned="true"
          logAbandoned="true"
          
          jdbcInterceptors="ConnectionState;StatementFinalizer;SlowQueryReportJmx(threshold=1000)"
          closeMethod="close" />
```

**Connection pool sizing formula:**
```
Pool Size = T × (C - 1)

Where T = number of threads (maxThreads)
      C = average number of DB connections per request
      
Example: 200 threads × 1 DB conn/request = 200 connections
But realistically: concurrent DB threads rarely exceed 10-20% of maxThreads.
```

---

## 🔹 Section 8: Testing, QA & Reliability

### Common Testing Patterns

**Unit test a Valve:**
```java
class RequestTimingValveTest {
    
    @Test
    void shouldDelegateToNextValve() throws Exception {
        var valve = new RequestTimingValve();
        var pipeline = new StandardPipeline();
        pipeline.setBasic(new DummyBasicValve());
        pipeline.addValve(valve);
        
        var req = createMockRequest("GET", "/test");
        var resp = createMockResponse();
        
        pipeline.getFirst().invoke(req, resp);
        
        // Verify next valve was invoked
        // Verify timing was logged
    }
}
```

**Integration test with embedded Tomcat:**
```java
class TomcatIntegrationTest {
    
    @Test
    void appDeploysAndResponds() throws Exception {
        var tomcat = new Tomcat();
        tomcat.setPort(0);  // random available port
        tomcat.addWebapp("/test", new File("src/test/webapp").getAbsolutePath());
        tomcat.start();
        
        var port = tomcat.getConnector().getLocalPort();
        var response = HttpClient.newHttpClient()
            .send(HttpRequest.newBuilder()
                    .uri(URI.create("http://localhost:" + port + "/test/health"))
                    .build(),
                  HttpResponse.BodyHandlers.ofString());
        
        assertEquals(200, response.statusCode());
        tomcat.stop();
    }
}
```

**Maven dependency for embedded Tomcat:**
```xml
<dependency>
    <groupId>org.apache.tomcat.embed</groupId>
    <artifactId>tomcat-embed-core</artifactId>
    <version>10.1.34</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.apache.tomcat.embed</groupId>
    <artifactId>tomcat-embed-jasper</artifactId>
    <version>10.1.34</version>
    <scope>test</scope>
</dependency>
```

### Reliability Checklist

| Concern | Configuration / Practice |
|---|---|
| Session stickiness | `jvmRoute` attribute on Engine; load balancer config |
| Graceful shutdown | Use `shutdown.sh`, not `kill -9` |
| Auto-reload | `reloadable="false"` in production; use blue/green deploy instead |
| OS limits | `ulimit -n 65536` (file descriptors), `ulimit -u 65536` (user processes) |
| Monitoring | Enable JMX: `-Dcom.sun.management.jmxremote` |
| Health checks | Endpoint returning `200 OK` — monitor `java.lang.Runtime.freeMemory/totalMemory` |
| Log rotation | AccessLogValve rotatable=true; use logrotate for catalina.out |
| Resource cleanup | `closeMethod="close"` on JNDI resources to avoid connection leaks on reload |

---

## 🔹 Section 9: The "Dark Arts" & War Stories (Expert Level)

### The 5 Most Notorious Tomcat Bugs

#### 1. `OutOfMemoryError: Metaspace` after N redeploys

**Root cause:** ClassLoader leak — the old WebappClassLoader is still referenced by a thread, JDBC driver, or static cache. Each reload clones the entire class tree in metaspace.

**Diagnosis:** Manager app → "Find Leaks" button. Or: `jmap -dump:live,format=b,file=leak.hprof <pid>` → Eclipse MAT → Search for `WebappClassLoader` with `started=false`.

**Fix:** 
```java
@WebListener
public class CleanupListener implements ServletContextListener {
    @Override
    public void contextDestroyed(ServletContextEvent sce) {
        // 1. Stop all custom threads
        // 2. Remove all ThreadLocal values
        // 3. Deregister JDBC drivers
        // 4. Shutdown executors
        // 5. Close all loggers
    }
}
```

#### 2. `java.net.SocketException: Too many open files`

**Root cause:** File descriptor leak — ZipFile handles from `unpackWARs=false` not closed, or NIO channel leak, or Unix socket not closed.

**Diagnosis:** 
```bash
lsof -p $(pgrep java) | wc -l          # count open FDs
lsof -p $(pgrep java) | grep -c REG     # file handles
lsof -p $(pgrep java) | grep -c TCP     # TCP sockets
```

**Common culprit:** `java.util.zip.ZipFile` handles from JAR scanning. Ensure `WEB-INF/lib` doesn't contain hundreds of JARs.

**Fix:** Raise `ulimit -n 65536`. Use `unpackWARs="true"` (ZipFile is closed after extract). Add `JarScanner scanManifest="false"` to context.xml.

#### 3. `Connection has already been closed` / `ClosedConnectionException`

**Root cause:** A JDBC connection was returned to the pool but was physically closed by the database (firewall timeout, DB restart, idle timeout).

**Fix:** Always configure `testOnBorrow=false` + `testWhileIdle=true` + `validationQuery`. Set `validationInterval` to avoid per-borrow validation overhead.

```xml
testOnBorrow="false"      <!-- don't test every borrow -->
testWhileIdle="true"      <!-- test idle connections in background -->
validationInterval="30000" <!-- max frequency 30s -->
```

#### 4. Session data lost after restart

**Root cause:** PersistentManager configured but serialization fails (non-serializable session attributes), or `pathname` for file store points to non-existent directory.

**Fix:**
```xml
<!-- Option 1: Disable persistence (accept session loss on restart) -->
<Manager pathname="" />

<!-- Option 2: Use external session store (Redis, Hazelcast) -->
<!-- Option 3: Use PersistentManager with proper serialization -->
<Manager className="org.apache.catalina.session.PersistentManager"
         saveOnRestart="true" maxIdleBackup="60">
    <Store className="org.apache.catalina.session.FileStore"
           directory="${catalina.base}/temp/sessions" />
</Manager>
```

**⚠️ All session attributes MUST implement `java.io.Serializable`** or `java.io.NotSerializableException` kills persistence.

#### 5. Intermittent `404` for a deployed application

**Root cause:** Race condition — application deployed but `web.xml` parsing not complete; or `ParallelWebappClassLoader` thread hasn't finished loading. Symptom: curl returns 200, but load balancer health check gets 404.

**Fix:** Set `deployOnStartup="true"` (not `false`) + `failCtxIfServletStartFails="true"`. Add a startup delay: `backgroundProcessorDelay="1"` (check every second). Or implement a proper health check Valve that verifies the Context is fully `STARTED`.

### Race Conditions, Deadlocks & Detection

**ThreadLocal + Thread Pool = cross-request data leak:**
```java
// WRONG — ThreadLocal survives across requests
private static ThreadLocal<User> currentUser = new ThreadLocal<>();

// FIX — always remove in finally
try {
    currentUser.set(user);
    // ... process request
} finally {
    currentUser.remove();  // ⚠️ CRITICAL
}
```

**Double-check this in:** Spring Security `SecurityContextHolder`, Log4J/MDC `ThreadContext`, any inheritable `ThreadLocal`.

### Security Pitfalls

#### 1. Manager App exposed to Internet

**Fix:** 
- Never expose `/manager/*` to the internet.
- Configure `RemoteCIDRValve` in `CATALINA_BASE/webapps/manager/META-INF/context.xml`:

```xml
<Context antiResourceLocking="false" privileged="true">
    <Valve className="org.apache.catalina.valves.RemoteCIDRValve"
           allow="127\.0\.0\.1|::1|10\.\d+\.\d+\.\d+" />
</Context>
```

#### 2. Default passwords / default users

**Fix:** Don't uncomment default users in `tomcat-users.xml`. Use strong passwords. Better: use LDAP/AD realm instead.

#### 3. Server information disclosure

**Fix:** Configure `ErrorReportValve` to hide server version:
```xml
<Valve className="org.apache.catalina.valves.ErrorReportValve"
       showReport="false" showServerInfo="false" />
```

#### 4. Directory listing enabled

**Fix:** In `web.xml`:
```xml
<servlet>
    <servlet-name>default</servlet-name>
    <servlet-class>org.apache.catalina.servlets.DefaultServlet</servlet-class>
    <init-param>
        <param-name>listings</param-name>
        <param-value>false</param-value>
    </init-param>
</servlet>
```

#### 5. SSL misconfiguration

**Fix:** 
```xml
<Connector port="8443" protocol="HTTP/1.1"
           SSLEnabled="true"
           scheme="https" secure="true">
    <SSLHostConfig>
        <Certificate certificateKeystoreFile="conf/keystore.jks"
                     certificateKeystorePassword="changeit"
                     type="RSA" />
    </SSLHostConfig>
</Connector>
```
- Use TLS 1.2+ only: `sslEnabledProtocols="TLSv1.2,TLSv1.3"`
- Disable weak ciphers

#### 6. CSRF on Manager App

The HTML Manager GUI has built-in CSRF protection. The Text and JMX Manager interfaces do NOT — never grant `manager-script` or `manager-jmx` roles to users with `manager-gui`.

### Observability

**JMX Monitoring "Big 6":**

| MBean | Attribute | What It Tells You |
|---|---|---|
| `java.lang:type=Memory` | `HeapMemoryUsage.used` | Current heap usage |
| `java.lang:type=Memory` | `HeapMemoryUsage.max` | Max heap allowed |
| `Catalina:type=ThreadPool,name="http-nio-8080"` | `currentThreadCount` | Current threads |
| `Catalina:type=ThreadPool,name="http-nio-8080"` | `currentThreadsBusy` | Active threads (should be < maxThreads) |
| `Catalina:type=GlobalRequestProcessor,name="http-nio-8080"` | `requestCount` | Total requests |
| `Catalina:type=GlobalRequestProcessor,name="http-nio-8080"` | `errorCount` | Total errors |

**JVM flags for JMX:**
```bash
CATALINA_OPTS="$CATALINA_OPTS
  -Dcom.sun.management.jmxremote
  -Dcom.sun.management.jmxremote.port=9090
  -Dcom.sun.management.jmxremote.ssl=false
  -Dcom.sun.management.jmxremote.authenticate=false
  -Djava.rmi.server.hostname=$(hostname -f)"
```

---

## 🔹 Section 10: The Ecosystem & Force Multipliers

### Top 5 Must-Know Third-Party Libraries

| Library | Purpose | Use Case |
|---|---|---|
| **Spring Boot** | Embed Tomcat, auto-configure, zero `server.xml` | 90%+ of new Tomcat deployments |
| **Apache HTTPD + mod_jk/mod_proxy** | Reverse proxy, SSL termination, static files | Production frontend for Tomcat |
| **Nginx** | Reverse proxy, rate limiting, static file serving | Lighter than Apache HTTPD |
| **Redis (for session storage)** | External session store (Spring Session, Redisson) | Distributed session in cluster |
| **JMXTrans / Prometheus + JMX Exporter** | Monitor Tomcat metrics | Observability into production |

### Comparison Matrix

| Feature | Nginx → Tomcat | Apache HTTPD → Tomcat | Tomcat Standalone |
|---|---|---|---|
| Static file perf | Excellent | Good | OK |
| SSL termination | Yes | Yes | Yes (JSSE/OpenSSL) |
| Load balancing | Built-in | mod_proxy_balancer | None (needs external) |
| Rate limiting | Built-in | mod_evasive | StuckThreadDetectionValve |
| Configuration | Simple | Complex (httpd.conf) | Medium (server.xml) |
| AJP support | No (HTTP only) | Yes (mod_proxy_ajp) | N/A |
| WebSocket proxy | Yes (1.13+) | Yes (mod_proxy_wstunnel) | Native |

### Cloud/Native Integrations

| Platform | Integration |
|---|---|
| **Docker** | Official `tomcat` image; mount `webapps/` and `conf/` |
| **Kubernetes** | Stateless Tomcat pods with external Redis for sessions; liveness/readiness probes |
| **AWS Elastic Beanstalk** | Pre-configured Tomcat platform; deploy WAR directly |
| **Azure App Service** | Java + Tomcat stack; deploy WAR |
| **Heroku** | Java buildpack detects and runs embedded Tomcat |
| **Google Cloud Run** | Build as container with embedded Tomcat |
| **Pivotal Cloud Foundry** | Java buildpack provisions Tomcat runtime |

### Spring Boot Embedded Tomcat — Key Properties

```properties
# application.properties — embedded Tomcat config

server.port=8080
server.tomcat.max-threads=400
server.tomcat.min-spare-threads=50
server.tomcat.accept-count=500
server.tomcat.max-connections=10000
server.tomcat.connection-timeout=20s
server.tomcat.max-http-header-size=32768
server.tomcat.uri-encoding=UTF-8
server.tomcat.accesslog.enabled=true
server.tomcat.accesslog.pattern=%h %l %u %t "%r" %s %b %D
server.tomcat.accesslog.directory=logs
server.tomcat.remoteip.remote-ip-header=x-forwarded-for
server.tomcat.remoteip.protocol-header=x-forwarded-proto
```

---

## 🔹 Section 11: The "Cheat Sheet" Quick Reference Cards

### CARD 1: Syntax Snippets for 80% of Daily Tasks

**server.xml — Minimal Production Minimal:**
```xml
<Server port="8005" shutdown="SHUTDOWN">
  <Listener className="org.apache.catalina.startup.VersionLoggerListener" />
  <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
  <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />
  <Service name="Catalina">
    <Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000" redirectPort="8443"
               maxThreads="400" acceptCount="500"  URIEncoding="UTF-8"
               enableLookups="false" compression="on" />
    <Engine name="Catalina" defaultHost="localhost">
      <Host name="localhost" appBase="webapps" unpackWARs="true" autoDeploy="false">
        <Valve className="org.apache.catalina.valves.AccessLogValve"
               directory="logs" prefix="access" suffix=".log"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
      </Host>
    </Engine>
  </Service>
</Server>
```

**JNDI DataSource — Quick Config:**
```xml
<!-- In context.xml (per-app) -->
<Resource name="jdbc/MyDB" auth="Container"
          type="javax.sql.DataSource"
          factory="org.apache.tomcat.jdbc.pool.DataSourceFactory"
          driverClassName="org.postgresql.Driver"
          url="jdbc:postgresql://host:5432/db"
          username="${db.user}" password="${db.pass}"
          maxActive="30" maxIdle="10" initialSize="5"
          testWhileIdle="true" validationQuery="SELECT 1"
          removeAbandonedTimeout="60" removeAbandoned="true"
          closeMethod="close" />
```

**Download & Run (no-install):**
```bash
wget -qO- https://dlcdn.apache.org/tomcat/tomcat-10/v10.1.34/bin/apache-tomcat-10.1.34.tar.gz | tar xz
mv apache-tomcat-10.1.34 tomcat
chmod +x tomcat/bin/*.sh
echo 'CATALINA_OPTS="-Xmx1g"' > tomcat/bin/setenv.sh
tomcat/bin/startup.sh
# → http://localhost:8080
tomcat/bin/shutdown.sh
```

### CARD 2: Terminal/CLI One-Liners

```bash
# Quick health check
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/

# Metrics snapshot
curl -s -u admin:pass http://localhost:8080/manager/text/serverinfo

# Thread dump (JDK 9+)
jhsdb jstack --pid $(pgrep java) | head -50

# Old thread dump
jstack -l $(pgrep java) > threaddump.txt

# GC stats every second
jstat -gcutil $(pgrep java) 1s

# Heap overview
jhsdb jmap --heap --pid $(pgrep java)

# Heap dump
jmap -dump:live,format=b,file=/tmp/heap.hprof $(pgrep java)

# Count active threads
jstack $(pgrep java) | grep -c "http-nio"

# Find leaked classloaders
jmap -clstats $(pgrep java) | grep WebappClass | head

# Deploy via curl
curl -u admin:pass --upload-file app.war "http://localhost:8080/manager/text/deploy?path=/app&update=true"

# Tail logs
tail -f logs/catalina.out logs/localhost_access_log.*.txt

# Find PID
ps aux | grep tomcat | grep java | awk '{print $2}'
```

### CARD 3: Configuration File Cheat Codes

**`catalina.properties` — Essential overrides:**
```properties
# Custom classloader paths (commented out by default)
server.loader=
shared.loader=

# Security
org.apache.catalina.connector.CoyoteAdapter.ALLOW_BACKSLASH=false
org.apache.catalina.connector.RECYCLE_FACADES=true
org.apache.tomcat.util.http.ServerCookie.ALLOW_HTTP_SEPARATORS_IN_V0=false
```

**`tomcat-users.xml` — Minimal production:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<tomcat-users xmlns="http://tomcat.apache.org/xml"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xsi:schemaLocation="http://tomcat.apache.org/xml tomcat-users.xsd"
              version="1.0">
  <!-- NO default users in production -->
  <!-- If you must have a manager user: -->
  <!--
  <role rolename="manager-gui"/>
  <user username="admin" password="CHANGE_ME_NOW" roles="manager-gui"/>
  -->
</tomcat-users>
```

**`setenv.sh` — JVM tuning template:**
```bash
#!/bin/sh
export CATALINA_OPTS="$CATALINA_OPTS \
  -Xms2g -Xmx4g \
  -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m \
  -XX:+UseZGC \
  -Djava.awt.headless=true \
  -Djava.security.egd=file:/dev/./urandom \
  -Dfile.encoding=UTF-8 \
  -Djava.net.preferIPv4Stack=true \
  -Dorg.apache.catalina.connector.RECYCLE_FACADES=true \
  -Dorg.apache.catalina.loader.WebappClassLoader.ENABLE_CLEAR_REFERENCES=true \
  -Xlog:gc*:file=${CATALINA_BASE}/logs/gc.log:time,uptime,level,tags"
```

### CARD 4: Error Messages & Their Exact Fixes

| Error Message | Root Cause | The Fix |
|---|---|---|
| `java.lang.OutOfMemoryError: Metaspace` | ClassLoader leak preventing GC of loaded classes | Find and fix classloader leak (threads, ThreadLocals, JDBC drivers). Increase MaxMetaspaceSize as temp fix. |
| `java.lang.OutOfMemoryError: Java heap space` | Heap exhausted — objects not GC'd | Increase -Xmx. Profile heap dump for leaks (sessions, caches). |
| `java.net.SocketException: Too many open files` | File descriptor leak (ZipFiles, sockets) | `lsof -p $(pgrep java) \| wc -l` to identify. Raise `ulimit -n`. Use `unpackWARs=true`. |
| `java.net.BindException: Address already in use` | Port conflict | Check `lsof -i :8080`. Kill existing process or change port. |
| `SEVERE: Error deploying web application directory [XXX]` | Corrupted WAR or missing web.xml | Re-deploy clean WAR. Check `catalina.YYYY-MM-DD.log` for full stack trace. |
| `org.apache.catalina.LifecycleException: Failed to start component [StandardEngine]` | Nested deployment failure — check the Caused by | Scroll up in logs for the actual cause (often a Context failing). |
| `SEVERE: The web application [XXX] created a ThreadLocal with key ... but failed to remove it` | ThreadLocal leak detected (logged, not thrown) | Fix the code to call `.remove()` in a finally block. |
| `java.lang.IllegalStateException: Cannot call sendError() after the response has been committed` | Response already committed (headers sent) before error | Flush output later. Check for unclosed `PrintWriter`/`OutputStream`. |
| `org.apache.catalina.connector.ClientAbortException: java.io.IOException: Broken pipe` | Client disconnected | Log at DEBUG only — not an error. Set `swallowAbortedUploads="true"`. |
| `HTTP 404 — Not found` | Wrong context path, missing servlet mapping, or app not deployed | Check `manager/text/list`. Verify `path` vs `docBase`. |
| `HTTP 405 — Method not allowed` | Servlet doesn't override `doGet()`/`doPost()` | Check servlet implementation. |
| `HTTP 500 — Error instantiating servlet class` | Missing no-arg constructor, or class not in classpath | Verify servlet has `public` no-arg constructor. Check WEB-INF/classes. |
| `org.apache.tomcat.jdbc.pool.ConnectionPool - Connection has been closed` | Physical DB connection closed but pool returned it | Configure `testWhileIdle=true` + `validationQuery`. Set `validationInterval`. |
| `java.sql.SQLException: No suitable driver` | JDBC driver not loaded | Put driver in `$CATALINA_HOME/lib` (or configure `$CATALINA_HOME/lib`). Or use `closeMethod="close"`. |
| `java.io.NotSerializableException` | Session attribute not Serializable | Make attribute class implement `Serializable`. Or disable session persistence: `Manager pathname=""`. |
| `ERROR: Exception Processing ErrorPage [errorPage]` | Error page itself has an error during error handling | Check the error page JSP/servlet. Add a plain-text fallback error page. |
| `WARNING: An attempt was made to expire a session with a negative remaining time` | Session expiry calculation overflow (clock skew) | Ensure NTP sync on all servers. Clock going backwards = immediate expiry. |
| `java.lang.SecurityException: Access denied` (SecurityManager) | Security policy blocks an operation | Check `catalina.policy` (Tomcat 10 and earlier). Fix policy or disable SM. |
| `HTTP 403 — Access to the requested resource has been denied` | Role/auth failure | Check `tomcat-users.xml` roles. Check `web.xml` security-constraint. |

---

> **Sources:** Apache Tomcat 10.1 Official Documentation (Architecture, Config Reference, Security How-To, ClassLoader How-To), Apache Tomcat Wiki (Memory, MemoryLeakProtection), CIS Apache Tomcat Benchmark, OpenLogic Security Best Practices, StackOverflow signal.
