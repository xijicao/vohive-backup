# syntax=docker/dockerfile:1

# 使用体积小的 Alpine；VoHive 为独立 Linux 二进制。
FROM alpine:3.22

# buildx 会自动传入 amd64 或 arm64，用来选择对应的原版文件。
ARG TARGETARCH

# ca-certificates 用于 HTTPS；tzdata 用于时区；qmi-utils 便于排查 QMI 模组。
RUN apk add --no-cache ca-certificates tzdata qmi-utils \
    && mkdir -p /app/config /app/data /app/logs

COPY linux/vohive_v1.5.5_linux_${TARGETARCH} /app/vohive
RUN chmod 0755 /app/vohive

WORKDIR /app

ENV TZ=Asia/Shanghai \
    CONFIG_PATH=/app/config/config.yaml

EXPOSE 7575

ENTRYPOINT ["/app/vohive"]
