# 🔐 Setting Up HTTPS-Based YUM Repositories on Oracle Linux 8

# Table of Contents
- [Repository Directory Structure](#repository-directory-structure)
- [SSL Certificate Deployment](#ssl-certificate-deployment)
- [Apache HTTPS Configuration](#apache-https-configuration)
- [Repository Metadata Creation](#repository-metadata-creation)
- [SELinux & File Permissions](#selinux--file-permissions)
- [Verification and Access Testing](#verification-and-access-testing)
- [Troubleshooting](#troubleshooting-reference)

This document outlines the steps required to configure a secure, internal YUM/DNF repository on Oracle Linux 8 using Apache with SSL, SELinux configuration, and STIG-aligned TLS enforcement.

<a id="repository-directory-structure"></a>
# 📁 Repository Directory Structure

The following directory structure was created under /var/www/html/repos to organize local repositories:

/var/www/html/repos/  
├── ol8_appstream_local  
├── ol8_baseos_local  
└── ol8_UEKR7_local  

File ownership and access controls:

chown -R root:root /var/www/html/repos  
chmod -R 755 /var/www/html/repos  
semanage fcontext -a -t httpd_sys_content_t "/var/www/html/repos(/.*)?"  
restorecon -Rv /var/www/html/repos  

<a id="ssl-certificate-deployment"></a>
# 🔐 SSL Certificate Deployment

SSL certificates can be self-signed or issued by an internal CA:  
openssl req -new -x509 -days 365 -nodes \  
-out /etc/pki/tls/certs/repo.crt \  
-keyout /etc/pki/tls/private/repo.key  

If provided with a .p7b CA chain:  
openssl pkcs7 -print_certs -in lab-ca-chain.p7b -out lab-ca-chain.pem  
cp lab-ca-chain.pem /etc/pki/ca-trust/source/anchors/  
update-ca-trust extract  

# Certificate verification:

openssl verify -CAfile /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem \  
/etc/pki/tls/certs/ol8-master-web.cer  

<a id="apache-https-configuration"></a>
# ⚙️ Apache HTTPS Configuration

Apache was configured to serve the repositories over HTTPS (port 8443) using STIG-aligned TLS configuration:  
/etc/httpd/conf.d/file-ssl.conf  

Listen 8443 https  

<details>
<summary>View Apache Configuration</summary>

```apache
<VirtualHost *:8443>
  ServerName ol8-master.lab.local
  DocumentRoot /var/www/html/files
  SSLEngine on
  SSLCertificateFile /etc/pki/tls/certs/ol8-master-web.cer
  SSLCertificateKeyFile /etc/pki/tls/private/ol8-master.lab.local.key.pem
  SSLCertificateChainFile /etc/pki/tls/certs/lab-ca-chain.pem
  SSLProtocol -all +TLSv1.2 +TLSv1.3
  SSLHonorCipherOrder on
  SSLCompression off
  SSLSessionTickets off

  <Directory "/var/www/html/files">
    Options +Indexes
    AllowOverride None
    Require all granted
  </Directory>

  ErrorLog /var/log/httpd/files_error.log
  CustomLog /var/log/httpd/files_access.log combined
</VirtualHost>
```

</details>

Apache was enabled and firewall rules applied:
systemctl enable --now httpd  
systemctl restart httpd  
firewall-cmd --permanent --add-service=https  
firewall-cmd --reload  

<a id=>"repository-metadata-creation"></>
# 🧰 Repository Metadata Creation

Install necessary tools and generate repo metadata:

dnf install -y createrepo yum-utils  
createrepo /var/www/html/repos/ol8_baseos_local/  
createrepo /var/www/html/repos/ol8_appstream_local/  

<a id="selinux--file-permissions"></a>
# 🧾 SELinux & File Permissions

The following script ensures proper SELinux context and file permissions:

#!/bin/bash  
chown -R apache:apache /var/www/html/files  
chmod -R o+rX /var/www/html/files  
restorecon -Rv /var/www/html/files  

Execute after updating repo content:  
/root/admintools/set_httpd_perms.sh  

🔍 Verification and Access Testing

<a id="verification-and-access-testing"></a>
# Verify access to the repository via curl:

curl -vk --cacert /etc/pki/tls/certs/lab-ca-chain.pem \  
https://<yourserver_fqdn>:8443/

Apache logs for troubleshooting:

tail -f /var/log/httpd/files_error.log  
tail -f /var/log/httpd/files_access.log

<a id="troubleshooting-reference"></a>
# 🧪 Troubleshooting Reference

Issue/Resolution

SSL handshake fails  
Confirm CA trust, cert paths, and curl -vk output

Access denied  
Inspect Apache Directory config and validate SELinux context

DNF can't reach repo  
Ensure correct metadata generation and .repo file configuration