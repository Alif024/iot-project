# Node-RED + HiveMQ + ngrok local stack

This repository contains a docker-compose configuration to run:

- HiveMQ CE as an MQTT broker (port 1883)
- Node-RED (port 1880) with a flow that exposes POST /api/publish
- nginx serving the `frontend/` folder as a static website (port 8081)
- ngrok container to expose the nginx site publicly (requires NGROK_AUTHTOKEN)

Quick start

1. Copy `.env.template` to `.env` and set your `NGROK_AUTHTOKEN`.
2. From the repo root run:

```
docker-compose --env-file .env up -d
```

3. Node-RED UI: http://localhost:1880/red (flows and logs at `./nodered`)
4. Static site: http://localhost:8081
5. ngrok will create a public URL in the ngrok container logs. To view logs:

```
docker logs -f ngrok
```

API

POST /api/publish

Payload: any JSON

Example:

```
curl -X POST http://localhost:1880/api/publish -H "Content-Type: application/json" -d '{"msg":"hello"}'
```

This will publish the JSON payload to MQTT topic `test/topic` on the HiveMQ broker.

Notes and assumptions

- You must supply a valid ngrok authtoken. See `.env.template`.
- The ngrok image used expects the command shown; if you prefer registering tunnels via dashboard or a different setup, update the `docker-compose.yml` accordingly.
- Node-RED flow is minimal; feel free to extend topics, QoS, authentication, or TLS as needed.