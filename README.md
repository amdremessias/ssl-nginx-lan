# ssl-nginx-lan
experimento-localhost-lan
SSL-
mkdir ~/certs_req
nano ~/certs_req/home.lan.com.req.cnf

//Colocar no ficheiro que se segue fazer as alterações necessárias:

[req]
distinguished_name = req_distinguished_name
x509_extensions = v3_req
prompt = no
[req_distinguished_name]
C = PT
ST = LX
L = LxCity
O = MinhaEmpresa
OU = Secção
CN = home.lan.com
[v3_req]
keyUsage = critical, digitalSignature, keyAgreement
extendedKeyUsage = serverAuth
subjectAltName = @alt_names
[alt_names]
DNS.1 = www.home.lan.com
DNS.2 = home.lan.com
DNS.3 = home.lan.com

//Depois de criado o ficheiro de requisitos req.cnf , podemos criar uma chave auto-assinada e o certificados com o OpenSSL numa única linha:

sudo openssl req -x509 -nodes -days 365 -newkey rsa:4096 -keyout /etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.crt -config ~/certs_req/home.lan.com.req.cnf -sha256

//Os dois ficheiros criados serão colocados nas sub-diretorias para o efeito “/etc/ssl/private” para a chave e .”/etc/ssl/certs” para o certificado.

Também devemos criar um forte grupo Diffie-Hellman, usado na negociação do Perfect Forward Secrecy com os clientes. isto vai demorar uns minutos dependendo da performance hardware que estiver na máquina.

sudo openssl dhparam -out /etc/nginx/dhparam.pem 4096

//E ficou assim concluído a criação do certificado SSL. A seguir é a configuração do Nginx para usar os ficheiros que acabamos de criar.
//1 – Criar um novo snippet (um troço de configuração usado para incorporar na configuração principal do servidor) de configuração apontado para a chave e para o certificado SSL.

sudo nano /etc/nginx/snippets/auto-assinado.conf

//Neste ficheiro “auto-assinado.conf” vamos colocar, com a diretiva “ssl_certificate” o certificado e a chave criados anteriormente.

ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;

//Depois de adicionado os ficheiro “auto-assinado.conf“, fazer ctrl+x em seguida <enter> para guardar na pasta e nome de ficheiro apresentado.
//2 – Criar mais um snippet de configuração com configurações de criptografia forte:

sudo nano /etc/nginx/snippets/ssl-params.conf

//Copiar o que está a seguir no seu ficheiro snippet “ssl-params.conf“:

ssl_protocols TLSv1.3;# Requires nginx >= 1.13.0 else use TLSv1.2
ssl_prefer_server_ciphers on; 
ssl_dhparam /etc/nginx/dhparam.pem; # openssl dhparam -out /etc/nginx/dhparam.pem 4096
ssl_ciphers EECDH+AESGCM:EDH+AESGCM;
ssl_ecdh_curve secp384r1; # Necessita nginx >= 1.1.0
ssl_session_timeout  10m;
ssl_session_cache shared:SSL:10m;
ssl_session_tickets off; # Necessita nginx >= 1.5.9
ssl_stapling on; # Necessita nginx >= 1.3.7
ssl_stapling_verify on; # Necessita nginx => 1.3.7
resolver $DNS-IP-1 $DNS-IP-2 valid=300s;
resolver_timeout 5s; 
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload";
add_header X-Frame-Options DENY;
add_header X-Content-Type-Options nosniff;
add_header X-XSS-Protection "1; mode=block";

//3 – Alterar a configuração do Nginx para usar SSL
//Antes de mais, melhor é fazer um backup das configurações do servidor a aplicar o SSL.
//No ficheiro de configuração existente, alterar as duas instruções “listen” para usar a porta 443 e SSL e, em seguida, inclua os dois //ficheiro de snippet que criamos anteriormente e criar um bloco de “server” para redirecionar http para https:

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    include snippets/auto-assinado.conf;
    include snippets/ssl-params.conf;

    //--eu editei diretamento no meu /etc/nginx/nginx.config 

#Lab_1_proxy0.0.1-m3ss14s-220402022
#raspizinho0
events {
        worker_connections 1024;
#<Tamanho max de conexoes simult. valor default 512
        }
        http {
                include mime.types;
#<Inclui MIME-Types para uso dos navegadores
                #include snippets/self-signed.conf;
                server {
                        listen 80;
                        listen 443 ssl;
                        server_name home.lan.com;
                        include snippets/auto-assinado.conf;
#                       include snippets/ssl-params.conf;
#                       ssl_certificate     /etc/nginx/ssl/server.crt;
#                       ssl_certificate_key /etc/nginx/ssl/server.key;
                        ssl_protocols       SSLv3 TLSv1 TLSv1.1 TLSv1.2;
                        ssl_ciphers         HIGH:!aNULL:!MD5;
#                       access_log /opt/agol_http.log my_log;
                        location ^~/bitwarden/ {
#< ^~/xxxxx/ =argumento para redirecionar para a porta do serviço correspondente
                                proxy_set_header Host home.lan.com;
#informar argumento no cabeçalho do =
                                proxy_pass http://192.168.5.77:8880/;
                                }
                        location ^~/tainer/ {
                                proxy_set_header Host home.lan.com;
                                proxy_pass https://192.168.5.77:9443/;
                                }
                        location ^~/snipe/ {
                                proxy_set_header Host home.lan.com;
                                proxy_pass http://192.168.5.77:8080/;
                                }
                        location ^~/webmin/ {
                                proxy_set_header Host home.lan.com;
                                proxy_pass http://192.168.5.77:10000/;
                                }
                        location ^~/urbackup/ {
                                proxy_set_header Host home.lan.com;
                                proxy_pass http://192.168.5.66:55414/;
                                }
                        location ^~/rabbitmq/ {
                                proxy_set_header Host home.lan.com;
                                proxy_pass http://192.168.5.77:15672/;
                                }

                        }
                        #root /etc/nginx/sites-enabled/reverse-proxy.conf;
        }
