# HTTP/2 Quick Reference Guide
## Cheat Sheet untuk Hands-on Lab

---

## 📌 Server Commands

```bash
# Start individual servers
npm run http1        # HTTP/1.1 on :8080
npm run http2        # HTTP/2 on :8443
npm run http2-push   # HTTP/2 + Push on :8444

# Start all servers at once
npm run all

# Run tests
npm test             # Basic connectivity test
npm run benchmark    # Performance comparison
```

---

## 🌐 URLs

| Server | URL | Features |
|--------|-----|----------|
| HTTP/1.1 | https://localhost:8080 | Multiple connections, text protocol |
| HTTP/2 | https://localhost:8443 | Multiplexing, binary, compression |
| HTTP/2 Push | https://localhost:8444 | All HTTP/2 + Server Push |

---

## 🔧 Testing Commands

### curl
```bash
# Test HTTP/2
curl -I --http2 https://localhost:8443

# Verbose output
curl -v --http2 https://localhost:8443

# Ignore certificate
curl -k --http2 https://localhost:8443
```

### nghttp
```bash
# Verbose HTTP/2 request
nghttp -nv https://localhost:8443

# Show frames
nghttp -v https://localhost:8443
```

### OpenSSL
```bash
# Test TLS connection
openssl s_client -connect localhost:8443 -alpn h2

# Check certificate
openssl x509 -in localhost.pem -text -noout
```

---

## 🎨 Chrome DevTools

### Network Tab Shortcuts
- `F12` - Open DevTools
- `Ctrl+Shift+P` - Command menu
- Type "protocol" - Show protocol column
- `Ctrl+R` - Reload
- `Ctrl+Shift+R` - Hard reload
- `Ctrl+Shift+Delete` - Clear cache

### Important Columns
- **Protocol**: Should show "h2" for HTTP/2
- **Connection ID**: Same for HTTP/2 (multiplexed)
- **Initiator**: Shows "Push / Other" for pushed resources
- **Timing**: Queueing should be minimal with HTTP/2

### Performance Tab
```javascript
// Get protocol
performance.getEntriesByType('navigation')[0].nextHopProtocol

// Get all resources
performance.getEntriesByType('resource')
```

---

## 🔍 Key Differences: HTTP/1.1 vs HTTP/2

| Feature | HTTP/1.1 | HTTP/2 |
|---------|----------|--------|
| **Protocol Type** | Text | Binary |
| **Connections** | 6-8 parallel | 1 multiplexed |
| **Request Queue** | Sequential | Concurrent |
| **Header Compression** | ❌ No | ✅ HPACK |
| **Server Push** | ❌ No | ✅ Yes |
| **Prioritization** | ❌ No | ✅ Yes |
| **HOL Blocking** | ✅ Yes | ❌ No (at HTTP layer) |

---

## 📊 HTTP/2 Frame Types

| Frame | Purpose | Common Use |
|-------|---------|------------|
| `DATA` | Carries body | Response content |
| `HEADERS` | HTTP headers | Request/response metadata |
| `PRIORITY` | Stream priority | Resource loading order |
| `RST_STREAM` | Terminate stream | Cancel request |
| `SETTINGS` | Connection params | Initial handshake |
| `PUSH_PROMISE` | Server push notification | Push CSS/JS |
| `PING` | Keepalive/measure RTT | Connection health |
| `GOAWAY` | Close connection | Graceful shutdown |
| `WINDOW_UPDATE` | Flow control | Manage data flow |
| `CONTINUATION` | Continue headers | Large headers |

---

## 🎯 HTTP/2 Concepts

### Streams
```
Connection
├── Stream 1 (HTML)
├── Stream 3 (CSS)   ← Concurrent!
└── Stream 5 (JS)    ← No blocking!
```

### Multiplexing
```
HTTP/1.1: [R1====][R2====][R3====]  ← Sequential
HTTP/2:   [R1][R2][R3][R1][R2][R3]  ← Interleaved
```

### HPACK Compression
```
Request 1: 400 bytes (full headers)
Request 2: 50 bytes  (87.5% compression!)
Request 3: 50 bytes
...
```

### Server Push
```
Client → Server: GET /index.html
Server → Client: 
  - PUSH_PROMISE /style.css
  - PUSH_PROMISE /app.js
  - DATA /index.html
  - DATA /style.css (pushed!)
  - DATA /app.js (pushed!)
```

---

## 🐛 Troubleshooting Quick Fixes

### Certificate Warning
```bash
# In Chrome: Type "thisisunsafe"
# Or regenerate:
rm localhost*.pem
./setup-http2-lab.sh
```

### Port Already in Use
```bash
# Find and kill process
lsof -ti:8443 | xargs kill -9
```

### Protocol Shows h1
```
- Use HTTPS, not HTTP
- Clear browser cache
- Check server file
- Restart server
```

### Module Not Found
```bash
npm install
```

### Server Won't Start
```bash
# Check ports
netstat -tuln | grep 8443

# Check Node.js
node --version  # Should be 16+
```

---

## 📈 Performance Metrics

### Expected Improvements
```
Multiplexing:     40-60% faster
Server Push:      2-3x fewer round trips
Header Compression: 85-90% bandwidth saved
Overall:          40-70% page load improvement
```

### Benchmark Results Format
```
HTTP/1.1:
  Total Time: 3000ms
  Requests/sec: 50

HTTP/2:
  Total Time: 1500ms  (50% faster!)
  Requests/sec: 100   (2x throughput!)
```

---

## 🔬 Experiment Checklist

### Before Each Experiment
- [ ] Clear browser cache
- [ ] Close other tabs
- [ ] Server is running
- [ ] DevTools Network tab open
- [ ] Protocol column enabled

### During Experiment
- [ ] Observe connection ID
- [ ] Check protocol column
- [ ] Note timing metrics
- [ ] Screenshot waterfall
- [ ] Check initiator column

### After Experiment
- [ ] Document results
- [ ] Compare with expected
- [ ] Note any anomalies
- [ ] Calculate improvements

---

## 📝 Test Results Template

```
Date: __________
Tester: __________

Experiment 1: Multiplexing
--------------------------
HTTP/1.1 Time: _____ ms
HTTP/2 Time:   _____ ms
Improvement:   _____ %
✓ Observed: Single connection / Multiple streams

Experiment 2: Server Push
--------------------------
Round Trips: HTTP/1.1: ___  HTTP/2: ___
✓ Observed: PUSH_PROMISE frames in DevTools

Experiment 3: Header Compression
--------------------------
Request 1: _____ bytes
Request 2: _____ bytes (compressed)
Savings:   _____ %

Overall Assessment:
- HTTP/2 is ___% faster on average
- Most impactful feature: ________________
- Recommendations: ____________________
```

---

## 💡 Pro Tips

### DevTools
- Enable "Disable cache" during tests
- Use "Preserve log" for multi-page analysis
- Filter by protocol: `protocol:h2`
- Export as HAR for detailed analysis

### Testing
- Run tests 3-5 times for average
- Clear cache between runs
- Close background apps
- Use stable network connection

### Debugging
- Check server console for logs
- Use `console.log()` liberally
- Test one feature at a time
- Verify basic HTTP/2 before advanced features

### Optimization
- Push only critical resources
- Don't over-push (wastes bandwidth)
- Set proper cache headers
- Monitor with real user data

---

## 🚀 Production Deployment

### Nginx
```nginx
server {
    listen 443 ssl http2;
    server_name example.com;
    
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
    
    # Push critical resources
    location = /index.html {
        http2_push /css/main.css;
        http2_push /js/app.js;
    }
}
```

### Caddy
```
example.com {
    # HTTP/2 enabled by default!
    tls cert.pem key.pem
    
    push /css/main.css
    push /js/app.js
}
```

### Node.js
```javascript
const http2 = require('http2');
const fs = require('fs');

const server = http2.createSecureServer({
  key: fs.readFileSync('key.pem'),
  cert: fs.readFileSync('cert.pem')
});

server.listen(443);
```

---

## 📚 Resources

### Official Specs
- RFC 7540: HTTP/2
- RFC 7541: HPACK
- https://http2.github.io/

### Tools
- Chrome DevTools
- Wireshark
- nghttp2
- h2load
- WebPageTest

### Learning
- Google Web Fundamentals
- MDN Web Docs
- "High Performance Browser Networking" book

---

## 🎓 Learning Path

1. ✅ Understand HTTP/1.1 limitations
2. ✅ Learn HTTP/2 concepts
3. ✅ Setup lab environment
4. ✅ Run experiments 1-6
5. ✅ Analyze performance gains
6. ✅ Document findings
7. 🎯 Implement in production
8. 🎯 Learn HTTP/3 next!

---

**Quick reminder:**
- HTTP/2 requires TLS in browsers
- Single connection = less overhead
- Multiplexing = no HOL blocking
- HPACK = compressed headers
- Server Push = proactive delivery
- Practice makes perfect! 🚀

---

*HTTP/2 Quick Reference v1.0*
*For HTTP/2 Hands-on Lab*
*Keep this handy during experiments!*
