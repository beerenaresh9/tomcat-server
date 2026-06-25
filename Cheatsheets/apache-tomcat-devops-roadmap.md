# Apache Tomcat — DevOps Learning Roadmap

> **Goal:** Everything a DevOps engineer needs to know about Apache Tomcat — from installation to production debugging.

---

## 📊 Mastery Levels

| Level | Label | What You Can Do |
|---|---|---|
| **L1** | Install & Run | Download, start/stop, deploy a WAR, check logs |
| **L2** | Configure | Tune connectors, JVM flags, datasources, realms |
| **L3** | Operate | Monitor, diagnose crashes, handle redeploys, manage sessions |
| **L4** | Architect | Clustering, load balancing, CI/CD pipelines, autoscaling |
| **L5** | Expert | Deep internals, classloader leak debugging, performance optimization, custom valves, embedding |

---

## 🔧 L1 — Install & Run (The Basics)

| # | Concept | Why It Matters for DevOps |
|---|---|---|
| 1.1 | **Download & Install** — binary vs package manager vs Docker | Must know how to provision Tomcat on any platform |
| 1.2 | **CATALINA_HOME vs CATALINA_BASE** | Home = binaries (shared), Base = instance-specific config. Enables multi-instance from one install. |
| 1.3 | **Startup/shutdown scripts** — `catalina.sh`, `startup.sh`, `shutdown.sh` | Automation, systemd service files, init scripts |
| 1.4 | **Directory structure** — `bin/`, `conf/`, `logs/`, `webapps/`, `work/`, `temp/` | Know where everything lives for troubleshooting |
| 1.5 | **Deploy a WAR** — manual copy, Manager app, curl | Fundamental deployment skill |
| 1.6 | **Access logs** — location, format, rotation | Audit, debugging, analytics |
| 1.7 | **Manager & Host-Manager apps** — roles, access control | Admin capability for deployments |
| 1.8 | **Ports** — 8080 (HTTP), 8443 (HTTPS), 8009 (AJP), 8005 (shutdown) | Firewall rules, security groups, port conflicts |

**Practical tasks:**
```
✓ Install Tomcat from tarball and start it
✓ Deploy a sample WAR (e.g., from manager/text/list)
✓ Check catalina.out for startup errors
✓ Stop and restart Tomcat
✓ Verify access logs are written
```

---

## ⚙️ L2 — Configuration (The Core)

### 2.1 `server.xml` Mastery

| # | Concept | Details |
|---|---|---|
| 2.1.1 | **Server element** — `port`, `shutdown` | The shutdown port — change from default `8005` in production |
| 2.1.2 | **Service element** | Groups connectors with engine; rarely changed |
| 2.1.3 | **Executor (thread pool)** — `maxThreads`, `minSpareThreads`, `maxIdleTime` | Controls request concurrency |
| 2.1.4 | **Connector attributes** — see table below | Every attribute matters for perf & security |
| 2.1.5 | **Engine** — `jvmRoute`, `defaultHost`, `backgroundProcessorDelay` | Clustering session stickiness |
| 2.1.6 | **Host** — `appBase`, `autoDeploy`, `unpackWARs`, `deployOnStartup` | Virtual host config, deployment behavior |
| 2.1.7 | **Context** — `path`, `docBase`, `reloadable`, `crossContext`, `sessionTimeout` | Per-application settings |
| 2.1.8 | **Valves** — `AccessLogValve`, `RemoteAddrValve`, `ErrorReportValve`, `StuckThreadDetectionValve` | Request pipeline interceptors |

**Connector attributes to know:**
```
maxThreads, minSpareThreads, acceptCount, maxConnections
connectionTimeout, keepAliveTimeout, maxKeepAliveRequests
enableLookups=false, compression=on, URIEncoding=UTF-8
maxHttpHeaderSize, maxPostSize, maxSwallowSize
```

### 2.2 JVM Configuration

| # | Concept | Details |
|---|---|---|
| 2.2.1 | **setenv.sh / setenv.bat** | Standard place to put JVM flags |
| 2.2.2 | **Heap sizing** — `-Xms`, `-Xmx` | Minimum and maximum heap |
| 2.2.3 | **Metaspace** — `-XX:MetaspaceSize`, `-XX:MaxMetaspaceSize` | Class metadata memory (replaces PermGen) |
| 2.2.4 | **GC selection** — G1, ZGC, Shenandoah | Latency vs throughput tradeoffs |
| 2.2.5 | **System properties** — `file.encoding=UTF-8`, `java.awt.headless=true`, `java.security.egd`, `preferIPv4Stack` | Prevent silent platform-dependent bugs |
| 2.2.6 | **GC logging** — `-Xlog:gc*` | Diagnose GC pauses, leak patterns |
| 2.2.7 | **JMX** — `-Dcom.sun.management.jmxremote` | Enable monitoring connectivity |

### 2.3 JNDI & Database Connectivity

| # | Concept | Details |
|---|---|---|
| 2.3.1 | **JDBC DataSource** — Tomcat JDBC Pool vs DBCP2 vs HikariCP | Connection pooling is critical for production |
| 2.3.2 | **Resource configuration** — `factory`, `driverClassName`, `url`, `username`, `password` | Standard JNDI datasource setup |
| 2.3.3 | **Connection pool tuning** — `maxActive`, `maxIdle`, `minIdle`, `maxWaitMillis` | Pool sizing for throughput |
| 2.3.4 | **Validation** — `testOnBorrow`, `testWhileIdle`, `validationQuery`, `validationInterval` | Prevent stale connection errors |
| 2.3.5 | **Leak detection** — `removeAbandonedTimeout`, `removeAbandoned`, `logAbandoned` | Catch unclosed connections |
| 2.3.6 | **closeMethod="close"** | Prevent JNDI connection pool leak on webapp reload |
| 2.3.7 | **JDBC driver placement** — `$CATALINA_HOME/lib` vs `WEB-INF/lib` | ClassLoader visibility → memory leaks |

**Practical tasks:**
```
✓ Configure a thread pool executor in server.xml
✓ Tune connector maxThreads + acceptCount
✓ Set up a JDBC datasource in context.xml
✓ Configure JVM heap (-Xms, -Xmx) in setenv.sh
✓ Enable JMX and verify MBeans are accessible
```

---

## 🚀 L3 — Operate (Day-to-Day Production)

### 3.1 Deployment

| # | Concept | Details |
|---|---|---|
| 3.1.1 | **WAR deployment methods** — Manager app, hot deploy, auto-deploy, CI/CD | Multiple paths to deploy |
| 3.1.2 | **`reloadable="true"` vs blue/green** | Hot reload is dev-only; never in production |
| 3.1.3 | **Parallel deployment** — deploy new version alongside old | Zero-downtime path |
| 3.1.4 | **CI/CD integration** — Jenkins/GitHub Actions deploy via Manager API | Automation pipeline |

### 3.2 Monitoring & Metrics

| # | Concept | Details |
|---|---|---|
| 3.2.1 | **Access logs** — combined vs common format, custom patterns | Audit trail, debugging |
| 3.2.2 | **JMX MBeans** — ThreadPool, GlobalRequestProcessor, Memory | Key metrics: currentThreadsBusy, errorCount, heap usage |
| 3.2.3 | **Prometheus + JMX Exporter** | Modern monitoring stack |
| 3.2.4 | **Health check endpoint** — custom Valve or status servlet | Load balancer probes |
| 3.2.5 | **Manager app serverinfo** — free memory, total memory, max memory | Quick pulse check |

### 3.3 Logging

| # | Concept | Details |
|---|---|---|
| 3.3.1 | **Log files** — catalina.out, catalina.YYYY-MM-DD.log, localhost.YYYY-MM-DD.log, access log | Know each file's purpose |
| 3.3.2 | **JULI logging** — Tomcat's java.util.logging fork | Default logging framework |
| 3.3.3 | **Logback/Log4j integration** | Replace default for production |
| 3.3.4 | **Log rotation** — AccessLogValve rotatable, logrotate for catalina.out | Prevent disk full |

### 3.4 Troubleshooting (The Essentials)

| # | Problem | Diagnosis Command | Root Causes |
|---|---|---|---|
| 3.4.1 | **Tomcat won't start** | `catalina.sh run` (foreground) | Port conflict, JVM flag error, missing CATALINA_HOME |
| 3.4.2 | **OutOfMemoryError** | `jstat -gcutil <pid> 1s`, heap dump analysis | Leaked classloaders, session bloat, cache growth |
| 3.4.3 | **Too many open files** | `lsof -p <pid> \| wc -l` | ZipFile handles, JDBC connections, sockets |
| 3.4.4 | **Slow responses** | Thread dump analysis (`jstack`) | DB pool exhaustion, lock contention, GC pauses |
| 3.4.5 | **Connection refused** | Check `acceptCount` + `maxConnections` + `netstat` | Thread pool full, backlog full, port not listening |
| 3.4.6 | **404 on deployed app** | `manager/text/list`, check docBase | Race condition, deployment failure, wrong path |
| 3.4.7 | **Session lost after restart** | Check Manager pathname, serialization | Non-serializable attributes, no persistence |

### 3.5 Session Management

| # | Concept | Details |
|---|---|---|
| 3.5.1 | **Session Manager** — StandardManager vs PersistentManager | In-memory vs persisted sessions |
| 3.5.2 | **Session persistence** — FileStore vs JDBCStore | Survive restart vs performance tradeoff |
| 3.5.3 | **Session timeout** — `web.xml` `<session-timeout>` vs context `sessionTimeout` | Which takes precedence |
| 3.5.4 | **Disable persistence** — `Manager pathname=""` | Stateless apps don't need it |

**Practical tasks:**
```
✓ Deploy a WAR via curl to manager/text/deploy
✓ Read a thread dump with jstack and identify stuck threads
✓ Enable Prometheus JMX Exporter to scrape Tomcat metrics
✓ Configure logrotate for catalina.out
✓ Diagnose a 404 by checking deployment state
```

---

## 🏗️ L4 — Architect (Clustering, CI/CD, Production Hardening)

### 4.1 Clustering & High Availability

| # | Concept | Details |
|---|---|---|
| 4.1.1 | **jvmRoute** — Engine attribute for load-balancer node ID | Sticky session routing |
| 4.1.2 | **Session replication** — DeltaManager (all-to-all) vs BackupManager (primary-standby) | Session HA across nodes |
| 4.1.3 | **External session store** — Redis, Hazelcast, Memcached (Spring Session) | Better than Tomcat's built-in replication |
| 4.1.4 | **Load balancer patterns** — nginx, Apache HTTPD, HAProxy | Reverse proxy + LB configuration |
| 4.1.5 | **AJP protocol** — Apache HTTPD + mod_proxy_ajp vs HTTP proxy | Binary protocol, SSL offload |
| 4.1.6 | **Sticky vs non-sticky sessions** | Stickiness avoids replication complexity |
| 4.1.7 | **Multiple instances on one host** — CATALINA_BASE separation | Resource isolation, port uniqueness |

### 4.2 CI/CD Pipeline Integration

| # | Concept | Details |
|---|---|---|
| 4.2.1 | **WAR build** — Maven `mvn package` / Gradle `gradle war` | Standard artifact format |
| 4.2.2 | **Automated deployment to Manager** — `curl --upload-file` | Script-based deployments |
| 4.2.3 | **Blue-green deployment** — deploy alongside, switch context | Zero-downtime |
| 4.2.4 | **Canary deployment** — deploy to subset of instances | Gradual rollout |
| 4.2.5 | **Rollback strategy** — keep previous WAR, undeploy new | Must-have for production |
| 4.2.6 | **Docker image** — build custom image with WAR baked in | Containerized deployment |

### 4.3 Security Hardening

| # | Concept | Details |
|---|---|---|
| 4.3.1 | **TLS/SSL configuration** — keystore, certificate, JSSE vs OpenSSL | End-to-end encryption |
| 4.3.2 | **Cipher suites & TLS versions** — disable SSLv3, TLS 1.0, TLS 1.1 | Security compliance |
| 4.3.3 | **Manager app lockdown** — RemoteCIDRValve, strong passwords, no default users | Critical — most attacked surface |
| 4.3.4 | **Server header hiding** — ErrorReportValve `showServerInfo="false"` | Reduce attack surface |
| 4.3.5 | **Directory listings off** — DefaultServlet `listings="false"` | Information leak prevention |
| 4.3.6 | **CORS filter** — restrict allowed origins | Cross-origin policy |
| 4.3.7 | **CSRF prevention filter** — especially for Manager | Cross-site request forgery |
| 4.3.8 | **HSTS, X-Frame-Options, content-type sniffing** — HttpHeaderSecurityFilter | Modern security headers |
| 4.3.9 | **Tomcat-users.xml** — locked down, passwords encrypted, LockOutRealm | Auth hardening |
| 4.3.10 | **Change shutdown port/password** — default `8005/SHUTDOWN` is a risk | Unauthorized shutdown prevention |
| 4.3.11 | **Remove unused connectors** — comment out AJP if not needed | Reduce attack surface |

### 4.4 Performance Architecture

| # | Concept | Details |
|---|---|---|
| 4.4.1 | **Reverse proxy pattern** — nginx/Apache in front of Tomcat | Static files, SSL, rate limiting, caching |
| 4.4.2 | **Static content offload** — serve from nginx/CDN, not Tomcat | Massive perf win |
| 4.4.3 | **Connection pooling** — right-sized DB pool | Prevent DB exhaustion |
| 4.4.4 | **GC tuning** — parallel vs concurrent GC | Tradeoff: throughput vs latency |
| 4.4.5 | **Thread pool sizing** — formula based on CPU cores + blocking coefficient | Right-size or waste resources |

### 4.5 Containerization & Cloud

| # | Concept | Details |
|---|---|---|
| 4.5.1 | **Official Tomcat Docker image** — layers, tags, versions | Base for custom images |
| 4.5.2 | **Dockerfile patterns** — multi-stage build, copy WAR to webapps | Clean images |
| 4.5.3 | **Kubernetes deployment** — probes, configmaps, statefulsets | Orchestration basics |
| 4.5.4 | **Readiness probe** — check context is up, not just port | Prevent routing to starting apps |
| 4.5.5 | **Liveness probe** — detect stuck threads, trigger restart | Self-healing |
| 4.5.6 | **Embedded Tomcat (Spring Boot)** — no server.xml, configure via properties | Modern alternative to standalone |

**Practical tasks:**
```
✓ Set up 2 Tomcat nodes with nginx load balancer
✓ Configure DeltaManager for session replication
✓ Write a GitHub Actions workflow to deploy WAR via Manager API
✓ Harden a production server.xml (disable AJP, hide server info, TLS)
✓ Create a Docker image with Tomcat + WAR + custom config
```

---

## 🧙 L5 — Expert (Deep Internals & Advanced Debugging)

### 5.1 ClassLoader Internals & Leak Debugging

| # | Concept | Details |
|---|---|---|
| 5.1.1 | **WebappClassLoader delegation model** — parent-first for EE APIs, parent-last for app | Understand class loading order |
| 5.1.2 | **Leak detection** — Manager "Find Leaks", heap dump analysis | Find leaked classloaders with `started=false` |
| 5.1.3 | **Leak chain anatomy** — `Registry → Class → ClassLoader → all classes → all static fields` | How one leak pins everything |
| 5.1.4 | **Common leak sources** — DriverManager, ThreadLocal, Timer threads, loggers, Introspector | Know the top offenders |
| 5.1.5 | **JreMemoryLeakPreventionListener** — how it works, what it fixes | Tomcat's built-in mitigation |
| 5.1.6 | **Thread renewal on reload** — how Tomcat 7+ mitigates ThreadLocal leaks | Why it works but code cleanup is better |

### 5.2 Pipeline-Valve Architecture

| # | Concept | Details |
|---|---|---|
| 5.2.1 | **Pipeline + Valve chain-of-responsibility** — Engine → Host → Context → Wrapper | Core architectural pattern |
| 5.2.2 | **BasicValve** — the terminal valve in each pipeline | StandardEngineValve, StandardHostValve, etc. |
| 5.2.3 | **Custom Valve development** — extend ValveBase, implement invoke() | Extend Tomcat behavior |
| 5.2.4 | **Valve vs Filter** — Valve is Tomcat-internal, Filter is per-webapp by spec | When to use which |

### 5.3 Connector Deep Dive

| # | Concept | Details |
|---|---|---|
| 5.3.1 | **NIO internals** — Acceptor → Poller → Worker thread handoff | Connection flow |
| 5.3.2 | **CoyoteAdapter** — bridges Coyote (network) to Catalina (container) | Adaptation layer |
| 5.3.3 | **HTTP/2 support** — h2c, h2, upgrade | Modern protocol |
| 5.3.4 | **NIO vs NIO2 vs APR** — thread models, performance characteristics | Connector selection |

### 5.4 Advanced Performance Tuning

| # | Concept | Details |
|---|---|---|
| 5.4.1 | **Flame graph generation** — async-profiler, perf + FlameGraph | CPU profiling |
| 5.4.2 | **Eclipse MAT** — dominator tree, GC roots, leak suspect report | Root cause analysis |
| 5.4.3 | **GC log analysis** — GCeasy, GCViewer | Pause time optimization |
| 5.4.4 | **Connection pool interceptor tuning** — SlowQueryReport, StatementFinalizer, ConnectionState | Sub-millisecond savings |
| 5.4.5 | **JVM tiered compilation** — `-XX:+TieredCompilation`, `-XX:CompileThreshold` | JIT warmup optimization |

### 5.5 Security Manager (Legacy — Tomcat 10 and earlier)

| # | Concept | Details |
|---|---|---|
| 5.5.1 | **catalina.policy** — grant permissions to webapps | Sandbox untrusted apps |
| 5.5.2 | **Default permissions** — what a standard webapp gets | Understand baseline |
| 5.5.3 | **Removed in Tomcat 11** — SecurityManager removed in Jakarta EE 11 | Plan accordingly |

**Practical tasks:**
```
✓ Fix a real classloader leak — reproduce via redeploy, detect with Eclipse MAT
✓ Write a custom Valve (e.g., request timing logger) and deploy it
✓ Generate and analyze a flame graph of a running Tomcat
✓ Debug a JVM crash (hs_err_pid.log) and identify the root cause
```

---

## 📋 DevOps Interview Quick Review

### Must-Know Commands

```bash
catalina.sh run                    # Start in foreground (debug)
catalina.sh start                  # Start as daemon
catalina.sh stop                   # Graceful stop
catalina.sh jpda start             # Debug mode (port 8000)
jstack <pid>                       # Thread dump
jmap -dump:live,format=b,file=heap.hprof <pid>  # Heap dump
jstat -gcutil <pid> 1s             # GC stats every second
jhsdb jmap --heap --pid <pid>      # Heap summary (JDK 9+)
lsof -p <pid> | wc -l              # File descriptor count
curl -u admin:pass "http://localhost:8080/manager/text/list"  # List apps
curl -u admin:pass --upload-file app.war "http://localhost:8080/manager/text/deploy?path=/app&update=true"  # Deploy
```

### Must-Know Configs

```bash
# server.xml — Connector
maxThreads, acceptCount, connectionTimeout, URIEncoding, compression

# setenv.sh — JVM
-Xms, -Xmx, -XX:MetaspaceSize, -XX:MaxMetaspaceSize
-XX:+UseZGC (or UseG1GC)

# context.xml — JDBC
maxActive, maxIdle, testWhileIdle, validationQuery, removeAbandoned

# web.xml — Security
http-only cookies, secure cookies, session-timeout, error-page
```

### Must-Know Architecture Interview Questions

| Question | Key Points |
|---|---|
| **How does Tomcat process a request?** | Connector → CoyoteAdapter → Engine Valve → Host Valve → Context Valve → Wrapper Valve → FilterChain → Servlet |
| **What causes classloader leaks?** | External objects (DriverManager, threads, JUL loggers) holding refs to webapp classes → metaspace OOM on redeploy |
| **How do you scale Tomcat?** | nginx L7 load balancer → multiple Tomcat nodes + Redis sessions (or sticky sessions + jvmRoute) |
| **How do you harden Tomcat for production?** | Disable AJP, hide server header, lockdown Manager, TLS 1.2+, RemoteCIDRValve, no default users, HSTS |
| **How do you tune Tomcat performance?** | Right-size thread pool, enable compression, use NIO connector, optimize JDBC pool, offload static assets |
| **What's the difference between Tomcat and Jetty/Undertow?** | Tomcat = full servlet spec compliance + JSP; Jetty = lighter, embedded-friendly; Undertow = non-blocking I/O |
| **What's in CATALINA_HOME vs CATALINA_BASE?** | HOME = shared binaries; BASE = instance-specific config. Run multiple instances from one HOME. |

---

## 🗺️ Recommended Learning Path

```
Week 1 — L1: Install & Run
  ├── Download, start/stop, deploy WAR, read logs
  └── Manager app basics

Week 2 — L2: Configuration
  ├── server.xml (connectors, executors, engines, hosts)
  ├── JVM flags (heap, GC, system properties)
  ├── JDBC datasources (pool config, validation, leak detection)
  └── Valves (access log, remote addr, error report)

Week 3 — L3: Operations
  ├── Monitoring (JMX, Prometheus, metrics)
  ├── Thread dump analysis
  ├── Heap dump + Eclipse MAT basics
  └── Session management

Week 4 — L4: Architecture
  ├── Clustering (jvmRoute, session replication, Redis)
  ├── Security hardening (TLS, Manager lockdown, headers)
  ├── CI/CD integration (WAR deploy, blue-green)
  └── Docker + Kubernetes deployment

Week 5+ — L5: Expert
  ├── ClassLoader leak debugging
  ├── Custom Valves
  ├── Performance optimization (profiling, GC tuning)
  └── Embedded Tomcat (Spring Boot internals)
```
