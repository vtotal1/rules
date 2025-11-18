# rules
Yara rules repository


#!/bin/bash
set -e

# Replace these with your actual IPs
ALLOWED_IP1="10.0.0.1"  # Replace with actual ip1
ALLOWED_IP2="10.0.0.2"  # Replace with actual ip2

echo "Installing required packages..."
dnf install -y iptables iptables-services rsyslog

echo "Stopping firewalld if running..."
systemctl stop firewalld 2>/dev/null || true
systemctl disable firewalld 2>/dev/null || true

echo "Enabling iptables services..."
systemctl enable iptables
systemctl enable ip6tables

echo "Setting up IPv4 rules..."

# Clear existing rules
iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X
iptables -t mangle -F
iptables -t mangle -X
iptables -t raw -F
iptables -t raw -X

# Set default policies
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Allow loopback
iptables -A INPUT -i lo -j ACCEPT

# Allow established connections
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow SSH from anywhere
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow all ports from specific IPs only
iptables -A INPUT -s $ALLOWED_IP1 -j ACCEPT
iptables -A INPUT -s $ALLOWED_IP2 -j ACCEPT

# Docker-specific rules
# Allow Docker bridge traffic from allowed IPs
iptables -A FORWARD -s $ALLOWED_IP1 -j ACCEPT
iptables -A FORWARD -s $ALLOWED_IP2 -j ACCEPT
iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT

# Docker NAT rules for allowed IPs
iptables -t nat -A DOCKER -s $ALLOWED_IP1 -j RETURN
iptables -t nat -A DOCKER -s $ALLOWED_IP2 -j RETURN

# Log dropped packets (rate limited)
iptables -A INPUT -m limit --limit 5/min -j LOG --log-prefix "IPT-DROP: " --log-level 4
iptables -A FORWARD -m limit --limit 5/min -j LOG --log-prefix "IPT-FWD-DROP: " --log-level 4

echo "Setting up IPv6 rules (block all)..."

# Clear existing IPv6 rules
ip6tables -F
ip6tables -X
ip6tables -t nat -F
ip6tables -t nat -X
ip6tables -t mangle -F
ip6tables -t mangle -X

# Block all IPv6
ip6tables -P INPUT DROP
ip6tables -P FORWARD DROP
ip6tables -P OUTPUT DROP

# Log dropped IPv6 packets
ip6tables -A INPUT -m limit --limit 5/min -j LOG --log-prefix "IP6T-DROP: " --log-level 4
ip6tables -A FORWARD -m limit --limit 5/min -j LOG --log-prefix "IP6T-FWD-DROP: " --log-level 4
ip6tables -A OUTPUT -m limit --limit 5/min -j LOG --log-prefix "IP6T-OUT-DROP: " --log-level 4

echo "Configuring rsyslog for iptables logging..."
cat > /etc/rsyslog.d/10-iptables.conf << 'EOF'
:msg, contains, "IPT-DROP:" -/var/log/iptables_dropped.log
:msg, contains, "IPT-FWD-DROP:" -/var/log/iptables_dropped.log
:msg, contains, "IP6T-DROP:" -/var/log/iptables_dropped.log
:msg, contains, "IP6T-FWD-DROP:" -/var/log/iptables_dropped.log
:msg, contains, "IP6T-OUT-DROP:" -/var/log/iptables_dropped.log
& stop
EOF

echo "Setting up log rotation..."
cat > /etc/logrotate.d/iptables << 'EOF'
/var/log/iptables_dropped.log {
    daily
    rotate 7
    missingok
    notifempty
    compress
    delaycompress
    postrotate
        /bin/kill -HUP `cat /var/run/rsyslogd.pid 2> /dev/null` 2> /dev/null || true
    endscript
}
EOF

echo "Restarting rsyslog..."
systemctl restart rsyslog

echo "Saving rules for persistence..."
iptables-save > /etc/sysconfig/iptables
ip6tables-save > /etc/sysconfig/ip6tables

echo "Starting iptables services..."
systemctl restart iptables
systemctl restart ip6tables

echo "Verifying Docker daemon settings..."
if command -v docker &> /dev/null; then
    mkdir -p /etc/docker
    if [ -f /etc/docker/daemon.json ]; then
        echo "Warning: /etc/docker/daemon.json exists. Verify iptables:true is set"
    else
        echo '{"iptables": true}' > /etc/docker/daemon.json
        systemctl restart docker || true
    fi
fi

echo "Current IPv4 rules:"
iptables -L -n -v

echo ""
echo "Setup complete!"
echo "Logs will be written to: /var/log/iptables_dropped.log"
echo "IMPORTANT: Replace ALLOWED_IP1 and ALLOWED_IP2 with your actual IPs!"
