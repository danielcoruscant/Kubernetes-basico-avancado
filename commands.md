## Comandos para certificado digital

    openssl req -x509 \
        -nodes \
        -days 365 \
        -newkey rsa:2048 \
        -keyout tls.key \
        -out tls.crt \
        -subj '/CN=*.demo.com/O=systemfactory' \
        -addext 'subjectAltName = DNS:*.demo.com'

    openssl x509 -in tls.crt -text

## Comandos para criação de secret

    kubectl create secret tls demo-secret --key tls.key --cert tls.crt
    
## Comandos utilitários

    echo -n "senhateste1234" | base64 
