sudo systemctl start redis-server        # Start Redis service
sudo systemctl stop redis-server         # Stop Redis service
sudo systemctl restart redis-server      # Restart Redis service (stop + start)
sudo systemctl reload redis-server       # Reload config without full restart (if supported)
sudo systemctl status redis-server       # Show Redis running status + logs summary
sudo systemctl enable redis-server       # Enable Redis to start automatically on boot
sudo systemctl disable redis-server      # Disable auto-start on boot
systemctl is-enabled redis-server        # Check if Redis auto-start is enabled or not
systemctl is-active redis-server         # Check if Redis is currently running or not

journalctl -u redis-server               # View Redis logs (history)
journalctl -u redis-server -f            # Follow Redis logs in real-time (live logs)

redis-cli                                # Open Redis command line interface
redis-cli ping                           # Test Redis connection (should return PONG)
redis-cli KEYS "*"                       # List all keys stored in Redis (not recommended in production)
redis-cli FLUSHALL                       # Delete ALL data from Redis (⚠️ destructive)
