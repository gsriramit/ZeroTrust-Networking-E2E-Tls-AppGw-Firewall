## Architecture Diagram
### TLS Flow
![NetworkSecurityNinja - Appgw-fw-e2e-tls](https://user-images.githubusercontent.com/13979783/210385376-310f8728-2d47-477e-8061-2badeb09ea96.png)

### Network Packet Flow
![NetworkSecurityNinja - Appgw-fw-packetflow](https://user-images.githubusercontent.com/13979783/210385277-a8e62633-13aa-416e-a68d-90d3c09efec2.png)


### TLS Flow (To be updated)
- A TLS certificate signed by a custom rootCA is attached to the Application gateway's listener. This should the certificate of the private CA that has signed the Intermediate CA certificate of Azure Firewall
- The client successfully establishes a TLS connection with the application gateway (only after the client accepts to proceed with the unverified connection. This has been detailed in the NOTES section)
- Application gateway being a L7 load balancer and a reverse proxy, terminates the TLS connection with the client
- As we have designed the connection to be TLS end to end, the application gateway now needs to establish a new/separate TLS connection with the target FQDN
- The UDR now routes the traffic through the azure firewall. Firewall has an application rule that *allows* the target URL and requires *TLS Inspection*
- Firewall at this point attempts to create a new TLS connection with the target site. It gets the server certificate of the target site during this process
- The firewall now uses its managed identity (MI) to access the Intermediate CA certificate from the keyvault to sign the *certificate that is created dynamically*. This would let the firewall create a certificate with the CN of the target site issued by the Firewall private Intermediate CA
- Firewall uses this certificate to complete the TLS connection with the application gateway. Firewall acts as a forward web proxy and impersonates the target site
- The customrootCA certificate that was used to sign the Intermediate CA certificate of the firewall has to be linked the backend HTTP settings of the app gateway
- Application Gateway, when it recives the server certificate from the firewall, would be able to verify the issuer of the cert as it has added the customrootCA to the list of trusted CAs
- In this architecture, the firewall acts as the client when interacting with the target site
- The target site uses the established TLS connection to respond back to the firewall which then sends the packets back to the appgw
- The appgw then terminates the TLS connection with the firewall and establishes a TLS connection with the client (browser) to send the response back

### Network Flow
Network traffic from the public internet follows this flow:

1. The client starts the connection with the public IP address of the Azure Application Gateway:
   - Source IP address: ClientPIP
   - Destination IP address: AppGwPIP
2. The request to the Application Gateway public IP is distributed to a back-end instance of the gateway, in this case 10.0.0.5.
   - This does not happen when the traffic needs to be sent to a external public site. The application gateway uses its public ip to send the traffic to the external fqdn
   - The Application Gateway would act as a reverse proxy. It terminates the connection with the client, and tends to establishe a new connection with the targeted FQDN.
   - The UDR to the targeted FQDN 192.0.78.24 & 192.0.78.25 in the Application Gateway subnet forwards the packet to the Azure Firewall i.e. 10.0.3.4, while preserving the destination IP of the ext.website
     - Source IP address: 10.0.0.5 (private IP address of the Application Gateway instance)
     - Destination IP address: 192.0.78.24
     - X-Forwarded-For header: ClientPIP
3. Azure firewall does 2 things in this step
   - It uses the configured DNS (Azure DNS in this case) to determine the public IP address of the target site as the host header does not contain the IP address but just the FQDN. If DNS is not configured in Firewall's DNS proxy settings, then this would break the flow 
   - It SNATs the traffic, because the traffic is going to a public IP address. It sends the traffic to the external website if the same is allowed by the configured application rules. Now that the traffic hits an application rule in the firewall, the target site will see the public IP address of the firewall, since the Azure Firewall will proxy the connection:
   - Source IP address if the traffic is allowed by an Azure Firewall application rule: 20.245.195.221 (public IP of the Azure Firewall)
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


## Deployment Instructions
- Run the CreateTLSCertificates.sh to generate the TLS files for the application gateway and Azure Firewall. Please note that this script will create all the certificates in the local machine from where the script is run from.  
- The images in the "DeploymentInstructions" directory contains the configuration settings of the Firewall & Firewall policy, application gateway and the route table

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
Since <yourblogspot.com> is not a publicly available domain, the DNS resolution of the same cannot happen at the public authoritative DNS servers or the Azure public DNS zone (if the management of the DNS has been delegated to the Azure public DNS zones). So, the mapping of the application gateway's public IP address to the domain name, i.e. contoso.com has to be done in the client's hosts file (<sysdrive>:\Windows\System32\drivers\etc\hosts in windows). This way, the client would still be able to send the requests to the application gateway even when the requests are sent to https://contoso.com  
example:  
**20.237.194.223 contoso.com**  
  
## Future Work
TBD
