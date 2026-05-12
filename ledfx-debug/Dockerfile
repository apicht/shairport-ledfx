# Diagnostic LedFx image: extends the official 2.1.8 release with paho-mqtt
# internal logging enabled, so we can see connect/disconnect/keepalive/reconnect
# events in LedFx logs.
#
# Background: ledfx/integrations/mqtt_hass.py constructs a paho-mqtt Client but
# never calls client.enable_logger(...). Without that call, paho-mqtt is silent
# regardless of Python's log level (-vv included). With it enabled, paho's
# internal events surface through LedFx's existing logger.
#
# Build (run on the deploy host, e.g. the QNAP):
#   docker build -t ledfx-debug:local ./ledfx-debug
#
# Use:
#   1. In .env: set LEDFX_IMAGE=ledfx-debug:local
#   2. docker compose up -d ledfx
#   3. Verify the patch took:
#        docker exec ledfx grep enable_logger \
#          /app/.venv/lib/python3.12/site-packages/ledfx/integrations/mqtt_hass.py
#      should print one line.
#   4. Watch logs for the next failure:
#        docker logs -f --timestamps ledfx 2>&1 \
#          | awk 'tolower($0) ~ /paho|mqtt/ { print; fflush() }' \
#          > /tmp/ledfx-paho.log &

ARG BASE_IMAGE=ledfxorg/ledfx:2.1.8
FROM ${BASE_IMAGE}

USER root

RUN set -e; \
    F=/app/.venv/lib/python3.12/site-packages/ledfx/integrations/mqtt_hass.py; \
    if [ ! -f "$F" ]; then \
        echo "FATAL: $F not found in base image" >&2; \
        exit 1; \
    fi; \
    if ! grep -q 'enable_logger(_LOGGER)' "$F"; then \
        sed -i '/^        client = mqtt\.Client()/a\        client.enable_logger(_LOGGER)' "$F"; \
    fi; \
    # Sanity: exactly one occurrence of the patch line.
    count=$(grep -c 'client.enable_logger(_LOGGER)' "$F"); \
    if [ "$count" != "1" ]; then \
        echo "FATAL: expected exactly 1 patch line, found $count" >&2; \
        exit 1; \
    fi; \
    echo "ledfx-debug: patched paho.enable_logger into mqtt_hass.py"

USER ledfx