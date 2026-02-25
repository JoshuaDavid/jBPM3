# Building jBPM 3 on Claude Code Web (gVisor sandbox)

This environment runs Ubuntu 24.04 on gVisor (Linux 4.4.0) with Java 21,
Maven 3.9.11, and no direct outbound DNS — all network traffic goes through
an authenticated HTTP proxy.

## Challenges

1. **No JDK 8 via apt** — DNS is not resolvable, so `apt-get install` fails.
   JDK 8 must be downloaded as a tarball via `curl` (which uses the proxy).
2. **Maven HTTP blocker** — Maven 3.8.1+ blocks `http://` repository URLs.
   The POM references `http://repository.jboss.org/...`. Must override the
   global settings to remove the blocker.
3. **Maven proxy auth** — Maven 3.9's native HTTP transport does not pick up
   proxy credentials from `JAVA_TOOL_OPTIONS`. Must use
   `-Dmaven.resolver.transport=wagon` for dependency resolution, and
   pre-download plugin artifacts via `curl` for plugin resolution.
4. **JDK 8 SSL trust store** — Temurin JDK 8's `cacerts` is outdated and
   fails TLS handshakes. Must copy JDK 21's `cacerts` to JDK 8.
5. **`javax.rmi.PortableRemoteObject`** — Removed in Java 11. Only
   `JndiUtil.java` uses it. JDK 8 is required to compile without code changes.
6. **`animal-sniffer-maven-plugin`** — Enforces Java 1.5 API compatibility.
   Must be skipped (`-Danimal.sniffer.skip=true`) since we run on JDK 8.

## Step-by-step instructions

### 1. Install JDK 8 (Temurin)

```bash
curl -L "https://github.com/adoptium/temurin8-binaries/releases/download/jdk8u482-b08/OpenJDK8U-jdk_x64_linux_hotspot_8u482b08.tar.gz" \
  -o /tmp/jdk8.tar.gz
mkdir -p /opt/jdk8
tar xzf /tmp/jdk8.tar.gz -C /opt/jdk8 --strip-components=1
rm /tmp/jdk8.tar.gz

# Fix SSL trust store (JDK 8's cacerts is outdated)
cp /usr/lib/jvm/java-21-openjdk-amd64/lib/security/cacerts \
   /opt/jdk8/jre/lib/security/cacerts
```

### 2. Configure Maven settings

Create `~/.m2/global-settings.xml` (overrides the default HTTP blocker):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.2.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.2.0 https://maven.apache.org/xsd/settings-1.2.0.xsd">
  <mirrors/>
</settings>
```

Create `~/.m2/settings.xml` with proxy credentials and a mirror to Maven Central:

```bash
PROXY_USER=$(echo "$JAVA_TOOL_OPTIONS" | grep -oP 'Dhttp.proxyUser=\K[^ ]+' | head -1)
PROXY_PASS=$(echo "$JAVA_TOOL_OPTIONS" | grep -oP 'Dhttp.proxyPassword=\K[^ ]+' | head -1)

cat > ~/.m2/settings.xml << XMLEOF
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.2.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.2.0 https://maven.apache.org/xsd/settings-1.2.0.xsd">
  <proxies>
    <proxy>
      <id>https-proxy</id>
      <active>true</active>
      <protocol>https</protocol>
      <host>21.0.0.165</host>
      <port>15004</port>
      <username>${PROXY_USER}</username>
      <password>${PROXY_PASS}</password>
      <nonProxyHosts>localhost|127.0.0.1</nonProxyHosts>
    </proxy>
  </proxies>
  <mirrors>
    <mirror>
      <id>central-mirror</id>
      <mirrorOf>*</mirrorOf>
      <name>Maven Central</name>
      <url>https://repo.maven.apache.org/maven2</url>
    </mirror>
  </mirrors>
</settings>
XMLEOF
```

**Note:** The proxy host/port (`21.0.0.165:15004`) and credentials are
environment-specific and may change between sessions. Always extract them
from `$JAVA_TOOL_OPTIONS` at the start of each session.

### 3. Pre-download Maven plugin dependencies

Maven's plugin resolution uses a transport that does not support the proxy
auth from `JAVA_TOOL_OPTIONS` or Maven settings. Dependencies (artifacts)
resolve fine with `-Dmaven.resolver.transport=wagon`, but plugins must be
pre-downloaded into `~/.m2/repository` via `curl`.

The following Python script recursively downloads all plugin POMs and JARs:

```bash
python3 << 'PYEOF'
import xml.etree.ElementTree as ET
import os, subprocess

BASE_URL = "https://repo.maven.apache.org/maven2"
REPO = os.path.expanduser("~/.m2/repository")
NS = {'m': 'http://maven.apache.org/POM/4.0.0'}
visited = set()

def download(group, artifact, version, ext="pom"):
    gpath = group.replace('.', '/')
    local_dir = os.path.join(REPO, gpath, artifact, version)
    fname = f"{artifact}-{version}.{ext}"
    local_file = os.path.join(local_dir, fname)
    if os.path.exists(local_file) and os.path.getsize(local_file) > 0:
        return True
    os.makedirs(local_dir, exist_ok=True)
    url = f"{BASE_URL}/{gpath}/{artifact}/{version}/{fname}"
    try:
        result = subprocess.run(["curl", "-sL", "-o", local_file, "-w", "%{http_code}", url],
                              capture_output=True, text=True, timeout=30)
        if result.stdout.strip() == "200" and os.path.getsize(local_file) > 0:
            return True
    except: pass
    if os.path.exists(local_file): os.remove(local_file)
    return False

def resolve(group, artifact, version, depth=0):
    if depth > 15: return
    key = f"{group}:{artifact}:{version}"
    if key in visited or not version or '$' in version or 'SNAPSHOT' in version: return
    visited.add(key)
    if not download(group, artifact, version, "pom"): return
    download(group, artifact, version, "jar")
    try:
        pf = os.path.join(REPO, group.replace('.', '/'), artifact, version, f"{artifact}-{version}.pom")
        tree = ET.parse(pf)
        root = tree.getroot()
        parent = root.find('m:parent', NS)
        if parent is not None:
            pg, pa, pv = (parent.findtext(f'm:{t}', namespaces=NS) for t in ['groupId','artifactId','version'])
            if pg and pa and pv: resolve(pg, pa, pv, depth+1)
        for dep in root.findall('.//m:dependency', NS):
            dg, da, dv = (dep.findtext(f'm:{t}', namespaces=NS) for t in ['groupId','artifactId','version'])
            scope = dep.findtext('m:scope', namespaces=NS) or 'compile'
            if dg and da and dv and '$' not in dv and 'SNAPSHOT' not in dv and scope != 'test':
                resolve(dg, da, dv, depth+1)
    except: pass

# Seed with all plugins needed for compile + test phases
seeds = [
    ("org.codehaus.mojo", "animal-sniffer-maven-plugin", "1.7"),
    ("org.apache.maven.plugins", "maven-surefire-plugin", "2.4.3"),
    ("org.apache.maven.surefire", "surefire-junit", "2.4.3"),
    ("org.apache.maven.plugins", "maven-compiler-plugin", "2.0.2"),
    ("org.apache.maven.plugins", "maven-resources-plugin", "2.4.1"),
    ("org.apache.maven.plugins", "maven-enforcer-plugin", "1.0-beta-1"),
    ("org.apache.maven.plugins", "maven-antrun-plugin", "1.3"),
    ("org.apache.maven.plugins", "maven-source-plugin", "2.0.4"),
    ("org.apache.maven.plugins", "maven-javadoc-plugin", "2.7"),
    # key transitive deps that the simple POM parser misses
    ("asm", "asm-all", "3.3.1"),
    ("org.apache.maven.doxia", "doxia-sink-api", "1.0-alpha-6"),
    ("doxia", "doxia-sink-api", "1.0-alpha-4"),
    ("org.codehaus.plexus", "plexus-utils", "1.2"),
]
for g, a, v in seeds:
    resolve(g, a, v)
print(f"Pre-downloaded {len(visited)} artifacts")
PYEOF
```

### 4. Compile the core module

```bash
JAVA_HOME=/opt/jdk8 mvn compile -f core/pom.xml -ntp \
  -gs ~/.m2/global-settings.xml \
  -s ~/.m2/settings.xml \
  -Danimal.sniffer.skip=true \
  -Dmaven.resolver.transport=wagon
```

### 5. Run the core test suite

```bash
JAVA_HOME=/opt/jdk8 mvn test -f core/pom.xml -ntp \
  -gs ~/.m2/global-settings.xml \
  -s ~/.m2/settings.xml \
  -Danimal.sniffer.skip=true \
  -Dmaven.resolver.transport=wagon
```

### 6. Clean up download failure markers (if retrying)

If a build fails partway through dependency resolution, Maven caches the
failure. Clear these before retrying:

```bash
find ~/.m2/repository -name "*.lastUpdated" -delete
```

## Test results (baseline)

With the above setup, the core module:

- **Compiles:** 407 source files, zero errors
- **Test-compiles:** 299 test source files, zero errors
- **Tests:** 1118 tests run
  - **1015 pass** (91%)
  - **10 failures + 93 errors** (all from ~21 test classes)
  - Root cause of most errors: `ConfigurationException: no info for object
    'resource.default.modules'` — these are tests that depend on
    `JbpmConfiguration` being initialized from `jbpm.cfg.xml`, which doesn't
    load properly in some test contexts on this platform. These same tests
    likely need Hibernate + HSQLDB and a full jBPM configuration context.
  - The passing 1015 tests include both unit tests and DB-backed integration
    tests (using in-memory HSQLDB).

## Quick reference: the one-liner

After initial setup (steps 1-3), compile + test in one shot:

```bash
find ~/.m2/repository -name "*.lastUpdated" -delete 2>/dev/null; \
JAVA_HOME=/opt/jdk8 mvn test -f core/pom.xml -ntp \
  -gs ~/.m2/global-settings.xml \
  -Danimal.sniffer.skip=true \
  -Dmaven.resolver.transport=wagon
```
