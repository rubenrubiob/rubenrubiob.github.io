```mermaid
flowchart TD
  domain[Domain]
  application[Application]
  infrastructure[Infrastructure]
  vendor[Vendor]
  
  application --> domain
  infrastructure --> application
  infrastructure --> domain
  infrastructure --> vendor
```