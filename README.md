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

Network traffic from the public internet follows this flow:

1. The client starts the connection to the public IP address of the Azure Application Gateway:
   - Source IP address: ClientPIP
   - Destination IP address: AppGwPIP
2. The request to the Application Gateway public IP is distributed to a back-end instance of the gateway, in this case 10.0.0.5.
   - This does not happen when the traffic needs to be sent to a external public site. The application gateway uses its public ip to send the traffic to the external fqdn
   - In this case, the Application Gateway instance stops the connection from the client, and tends to establishe a new connection with the targeted FQDN.
   - The UDR to 192.0.78.24 & 192.0.78.25 in the Application Gateway subnet forwards the packet to the Azure Firewall i.e. 10.0.3.4, while preserving the destination IP of the ext.website
     - Source IP address: 10.0.0.5 (private IP address of the Application Gateway instance)
     - Destination IP address: 192.0.78.24
     - X-Forwarded-For header: ClientPIP
3. Azure Firewall does SNAT the traffic, because the traffic is going to a public IP address. It forwards the traffic to the external website if the same is allowed by the configured application rules. Now that the traffic hits an application rule in the firewall, the workload will see the source IP address of the specific firewall instance that processed the packet, since the Azure Firewall will proxy the connection:
   - Source IP address if the traffic is allowed by an Azure Firewall application rule: 20.245.195.221
   - Destination IP address: 192.0.78.24
   - X-Forwarded-For header: ClientPIP
4. The ext website answers the request, reversing source and destination IP addresses.
   - Source IP address: 192.0.78.24
   - Destination IP address: 20.245.195.221
5. Here the Azure Firewall doesn SNAT the traffic again as the traffic needs to be sent to the private IP of the application gateway
   - Source IP address: 10.0.3.4
   - Destination IP address: 10.0.0.5
6. Finally, the Application Gateway instance answers the client:
   - Source IP address: AppGwPIP
   - Destination IP address: ClientPIP  
Outbound flows from the VMs to the public internet go through Azure Firewall, as defined by the UDR to 0.0.0.0/0. 

## Deployment Instructions
- Run the CreateTLSCertificates.sh to generate the TLS files for the application gateway and the backend server. Please note that this script will create all the certificates in the local machine from where the script is run from
  - If you require the certificates to be loaded to an instance of Azure Keyvault, you can do the same and create a reference to the cert for App gateway to use. The instructions for the same are available in this article- https://learn.microsoft.com/en-us/azure/application-gateway/key-vault-certs
- Modify the variables according to your azure environment settings in the DeployAzureResources.ps1 file. The commands can be run one step at a time if you are getting familiar with the behavior of each of the commands. The ps1 file should also work when executed from any automation pipeline
- The server certificate created for the backend virtual machine(s) should be installed manually in this exercise. The instructions to complete the procedure in IIS 7.0 have been made available in post referenced in the next section. The procedure should be similar to other web servers too, including apache tomcat and Nginx
  - The more elegant way of doing this is to have the server TLS certificates baked into the virtual machine images and have these images used from the Compute Image gallery. However, these images should not be used for common workloads. If the installed certificates are exportable and the password has not been secured, then that is a potential security risk
  - The certificates could also be read from the keyvault (using the managed identity of the VM being deployed) and installed in the certificate store. This process could be completed as a part of the post deployment automation script

## Implementation References

Enabling Zero-Trust with Azure Networking Services
https://azure.microsoft.com/en-us/blog/enabling-zero-trust-with-azure-network-security-services/

Implementation of e2e TLS with Application Gateway and Azure Firewall
https://techcommunity.microsoft.com/t5/azure-network-security-blog/zero-trust-with-azure-network-security/ba-p/3668280

Azure Firewall Premium SKU- Configuration of certificates for TLS inspection
https://learn.microsoft.com/en-us/azure/firewall/premium-certificates

Microsoft Security Community Webinar on this concept
https://www.youtube.com/watch?v=JFEA-M0YDMY&ab_channel=MicrosoftSecurityCommunity

Implementation of Zero-Trust in various topologies
https://learn.microsoft.com/en-us/azure/architecture/example-scenario/gateway/application-gateway-before-azure-firewall  
- Client from public connections accessing the workloads through appgw
- Clients from connected networks (hybrid: on-premise connections)
- Azure Firewall in a azure vWan making it a **Secure Hub** 
- Using a 3rd party NVA with Route Server

Network packet flow in a topology with appgw before firewall
https://learn.microsoft.com/en-us/azure/architecture/example-scenario/gateway/firewall-application-gateway#application-gateway-before-firewall

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
