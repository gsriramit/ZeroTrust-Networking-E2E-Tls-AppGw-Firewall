## Architecture Diagram
TBD

### TLS Flow
- A TLS certificate signed by a custom rootCA is attached to the Application gateway's listener. The Listener has a frontend IP configuration consisting of the public IP and listening on port 443
- The client successfully establishes a TLS connection with the application gateway (only after the client accepts to proceed with the unverified connection. This has been detailed in the NOTES section)
- Application gateway being a L7 load balancer and a reverse proxy, terminates the TLS connection with the client
- As we have designed the connection to be TLS end to end, the application gateway now needs to establish a new/separate TLS connection with the backend server
- The backend server has a TLS certificate installed and added to the default website (CN= contoso.com) signed by the same custom rootCA 
- In this part of the connection, the application gateway acts as the client and the workload machine acts as the server
  - For the application gateway to successfully establish a TLS connection with the server, the server's CA signing certificate has to be added to the backend HTTP settings of the app gateway. This configuration instructs the app gateway to trust the server's TLS certificate
  - Note: This step is not required if the backend's certificate is signed by a well-known CA such as Verisign. The same is applicable to Azure services such as app services too
- The backend server uses the established TLS connection to respond back to the appgw
- The appgw then terminates the TLS connection with the server and establishes a TLS connection with the client (browser)

### Network Flow

1. The client starts the connection to the public IP address of the Azure Application Gateway:
   - Source IP address: ClientPIP
   - Destination IP address: AppGwPIP
2. The request to the Application Gateway public IP is distributed to a back-end instance of the gateway, in this case 10.0.0.7. The Application Gateway instance that receives the request stops the connection from the client, and establishes a new connection with one of the back ends. The back end sees the Application Gateway instance as the source IP address. The Application Gateway inserts an X-Forwarded-For HTTP header with the original client IP address.
   - Source IP address: 10.0.0.7 (the private IP address of the Application Gateway instance)
   - Destination IP address: 10.0.2.4
   - X-Forwarded-For header: ClientPIP
3. The VM answers the application request, reversing source and destination IP addresses. The VM already knows how to reach the Application Gateway, so doesn't need a UDR.
   - Source IP address: 10.0.2.4
   - Destination IP address: 10.0.0.7
4. Finally, the Application Gateway instance answers the client:
   - Source IP address: AppGwPIP
   - Destination IP address: ClientPIP  
Azure Application Gateway adds metadata to the packet HTTP headers, such as the **X-Forwarded-For** header containing the original client's IP address. Some application servers need the source client IP address to serve geolocation-specific content, or for logging. For more information, see How an application gateway works.  

The IP address 10.0.0.7 is one of the instances the Azure Application Gateway service deploys under the covers, here with the internal, private front-end IP address 10.0.0.4. These individual instances are normally invisible to the Azure administrator. But noticing the difference is useful in some cases, such as when troubleshooting network issues.  

The flow is similar if the client comes from an on-premises network over a VPN or ExpressRoute gateway. The difference is the client accesses the private IP address of the Application Gateway instead of the public address.  

## Deployment Instructions
- Run the CreateTLSCertificates.sh to generate the TLS files for the application gateway and the backend server. Please note that this script will create all the certificates in the local machine from where the script is run from
  - If you require the certificates to be loaded to an instance of Azure Keyvault, you can do the same and create a reference to the cert for App gateway to use. The instructions for the same are available in this article- https://learn.microsoft.com/en-us/azure/application-gateway/key-vault-certs
- Modify the variables according to your azure environment settings in the DeployAzureResources.ps1 file. The commands can be run one step at a time if you are getting familiar with the behavior of each of the commands. The ps1 file should also work when executed from any automation pipeline
- The server certificate created for the backend virtual machine(s) should be installed manually in this exercise. The instructions to complete the procedure in IIS 7.0 have been made available in post referenced in the next section. The procedure should be similar to other web servers too, including apache tomcat and Nginx
  - The more elegant way of doing this is to have the server TLS certificates baked into the virtual machine images and have these images used from the Compute Image gallery. However, these images should not be used for common workloads. If the installed certificates are exportable and the password has not been secured, then that is a potential security risk
  - The certificates could also be read from the keyvault (using the managed identity of the VM being deployed) and installed in the certificate store. This process could be completed as a part of the post deployment automation script

## Implementation References

Network Architecture & packet flow with Application Gateway and a backend server  
https://learn.microsoft.com/en-us/azure/architecture/example-scenario/gateway/firewall-application-gateway#architecture-1

End-to-End TLS Encryption in the app gateway setup
https://learn.microsoft.com/en-us/azure/application-gateway/ssl-overview#end-to-end-tls-encryption

Configuring E2E TLS in an app gateway setup using powershell
https://learn.microsoft.com/en-us/azure/application-gateway/application-gateway-end-to-end-ssl-powershell

Generating self-signed certificates for the backend workload
https://learn.microsoft.com/en-us/azure/application-gateway/self-signed-certificates

IIS
For instructions on how to import certificate and upload them as server certificate on IIS, see HOW TO: Install Imported Certificates on a Web Server in Windows Server 2003(https://support.microsoft.com/help/816794/how-to-install-imported-certificates-on-a-web-server-in-windows-server)

For TLS binding instructions, see How to Set Up SSL on IIS 7(https://learn.microsoft.com/en-us/iis/manage/configuring-security/how-to-set-up-ssl-on-iis#create-an-ssl-binding-1)

Converting a cert file into a cer file
https://support.comodo.com/index.php?/Knowledgebase/Article/View/361/17/how-do-i-convert-crt-file-into-the-microsoft-cer-format

## Note:
Since contoso.com is not a publicly available domain, the DNS resolution of the same cannot happen at the public authoritative DNS servers or the Azure public DNS zone (if the management of the DNS has been delegated to the Azure public DNS zones). So, the mapping of the application gateway's public IP address to the domain name, i.e. contoso.com has to be done in the client's hosts file (<sysdrive>:\Windows\System32\drivers\etc\hosts in windows). This way, the client would still be able to send the requests to the application gateway even when the requests are sent to https://contoso.com  
example:  
**20.237.194.223 contoso.com**  
  
## Future Work
*Designing end-to-end TLS for workloads that run on AKS pods using Application gateway Inress Controller (AGIC)*  
**Note**: In this case the CA certificate that is added to the backend HTTP settings should be the certificate of the authority that signed the TLS certificates of the workload pods 
