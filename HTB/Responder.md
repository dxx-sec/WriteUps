# Responder
- 10.129.162.110 -> nos redirige a unika.htb
- Asociamos el DNS local de la ip a -> unika.htb
- p-80 web 5985 -> WinRM y 7680 pando-pub

- En la web vemos que hay un parametro para cambiar el idioma y podemos realizar LFI

- http://unika.htb/index.php?page=../../../../../../../../windows/system32/drivers/etc/hosts
