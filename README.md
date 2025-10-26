## HOW TO RUN

create a bash script:

```test.sh
#!/bin/bash
echo "=== Testing Blue/Green Failover ==="

echo -e "\n1️⃣  Testing normal state (should be Blue)..."
for i in {1..3}; do
  curl -s http://localhost:8080/version | grep -o '"pool":"[^"]*"'
done

echo -e "\n\n2️⃣  Triggering chaos on Blue..."
curl -X POST http://localhost:8081/chaos/start?mode=error

echo -e "\n\n3️⃣  Testing failover (should switch to Green)..."
sleep 2
for i in {1..5}; do
  echo -n "Request $i: "
  curl -s http://localhost:8080/version | grep -o '"pool":"[^"]*"'
  sleep 0.5
done

echo -e "\n\n4️⃣  Stopping chaos..."
curl -X POST http://localhost:8081/chaos/stop

echo -e "\n\n✅ Test complete!"
```