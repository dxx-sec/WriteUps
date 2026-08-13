# Redeemer

- Lanzamos escaneo y nmap -sCV -p- 10.129.156.202
- 6379/tcp open  redis   Redis key-value store 5.0.7
- redis-cli para conectarse  -> -h hostname
- info para pillar info del server una vez conectados.
- SELECT -> db -> KEYS para ver todas las claves ( clave - valor)
- GET Para consultar una clave en nuestro caso -> flag (2) y nos da la flag.
