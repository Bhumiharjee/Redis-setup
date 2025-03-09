# Redis Deployment using GitLab CI/CD

## Overview
This project automates the deployment of Redis on an Ubuntu-based environment using GitLab CI/CD. The pipeline ensures that Redis is installed, configured, and accessible externally.

## Features
- Automatically updates system packages.
- Installs and configures Redis.
- Enables external access to Redis.
- Opens firewall ports for Redis.
- Validates Redis service status and connectivity.

## Deployment Pipeline
The deployment is handled via GitLab CI/CD with the following steps:

1. **Update System Packages**: Ensures the latest updates are installed.
2. **Install Redis**: Installs Redis server on Ubuntu.
3. **Configure Redis**: Enables Redis as a service and starts it.
4. **Allow External Access**: Modifies Redis configuration to allow remote connections.
5. **Open Firewall Ports**: Updates the firewall settings to allow external access.
6. **Connectivity Tests**: Verifies internal and external Redis accessibility.

## GitLab CI/CD Configuration (`.gitlab-ci.yml`)
```yaml
image: ubuntu:latest

stages:
  - deploy

deploy:
  stage: deploy
  tags:
    - hurry-solv-dev-svc
  script:
    - |
      echo "🔄 Updating system packages..."
      sudo apt update && sudo apt upgrade -y
      echo "✅ System updated."

      echo "🚀 Installing Redis..."
      sudo apt install -y redis-server
      echo "✅ Redis installed."

      echo "🔧 Configuring Redis..."
      sudo systemctl enable redis-server
      sudo systemctl start redis-server
      echo "✅ Redis started."

      echo "📝 Checking Redis status..."
      if systemctl is-active --quiet redis-server; then
        echo "✅ Redis is running."
      else
        echo "❌ Redis is NOT running!"
        journalctl -u redis-server --no-pager | tail -n 20
        exit 1
      fi

      echo "🌍 Fetching external IP..."
      EXTERNAL_IP=$(curl -s ifconfig.me)
      REDIS_PORT=6379
      echo "🚀 Redis is accessible at: redis://$EXTERNAL_IP:$REDIS_PORT"

      echo "🔧 Allowing external access to Redis..."
      sudo sed -i 's/^bind 127.0.0.1 ::1/# bind 127.0.0.1 ::1/' /etc/redis/redis.conf
      sudo sed -i 's/^protected-mode yes/protected-mode no/' /etc/redis/redis.conf
      sudo systemctl restart redis-server
      echo "✅ Redis is now accessible from outside."

      echo "🛡️ Opening firewall for Redis..."
      sudo ufw allow 6379/tcp
      sudo ufw reload || echo "Firewall reload skipped"
      echo "✅ Firewall updated."

      echo "🔍 Testing external Redis connectivity..."
      if nc -zv $EXTERNAL_IP $REDIS_PORT; then
        echo "✅ Redis is reachable externally at redis://$EXTERNAL_IP:$REDIS_PORT"
      else
        echo "❌ Redis is NOT reachable externally. Check firewall and network settings."
      fi

      echo "🔍 Testing internal Redis connectivity..."
      if nc -zv 127.0.0.1 $REDIS_PORT; then
        echo "✅ Redis is reachable internally at 127.0.0.1:$REDIS_PORT"
      else
        echo "❌ Redis is NOT reachable internally! Check service configuration."
        exit 1
      fi
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: always
    - if: $CI_COMMIT_TAG
      when: always
```

## Prerequisites
- A GitLab runner configured with appropriate tags (`hurry-solv-dev-svc`).
- Firewall rules allowing access to port `6379`.
- Ubuntu-based system.

## Usage
1. Push changes to the repository.
2. GitLab CI/CD will trigger the deployment.
3. Redis will be installed, configured, and made accessible externally.

## Verification
To verify Redis is running correctly:
```sh
redis-cli -h <EXTERNAL_IP> -p 6379 ping
```
Expected output:
```sh
PONG
```

## Security Considerations
- This setup disables Redis protected mode, allowing external access.
- Ensure the firewall and security groups restrict access to authorized users.
- Consider setting up authentication for Redis (`requirepass <password>` in `redis.conf`).

## License
This project is licensed under the MIT License.

## Author
[Saurav](https://github.com/bhumiharjee)

