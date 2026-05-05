# Create a self-signed certificate
```sh
apt-get update
apt-get install openssl
openssl req -x509 -newkey rsa:4096 -keyout privkey.pem -out fullchain.pem -sha256 -days 365 -nodes
```

# Create the directory if it doesn't exist
```sh
sudo mkdir -p /etc/ssl/certs/
```
# Move the files
```sh
sudo mv fullchain.pem /etc/ssl/certs/
sudo mv privkey.pem /etc/ssl/certs/
```
# Set permissions (Important for security!)
```sh
sudo chmod 600 /etc/ssl/certs/privkey.pem
sudo chmod 644 /etc/ssl/certs/fullchain.pem
```