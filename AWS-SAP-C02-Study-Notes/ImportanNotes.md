# WAF
-Purpose 
 - Block
   - SQL Injection
   - Cross Site Scripting 
 - In Fronf Of
   - ELB
   - Cloud Front
   - 
 - Rule
   - Type
      - Geo
      - IP Set 
      - Header
   - Action
     - Allow
     - Block 
# Auto Scaling
  - Flow 
    - EC2 Cloud Watch Agent
    - CLoud Watch 
    - Auto Scaling Group
      - Take Action based on Metrics
    - Type
      - Dyanmic Tracking
      - Step Scaling
      - Scheduled Scaling
         - Desired Count
         - Min 
         - Max
    - Scaling Settings
      - Cooldowns
        - 300 seconds
        - no scale up and down 
      - Termination Policy 
         - terminate first
      - Termination Protection
         - protected instances
      - Standby State

# ALB 
  - Host
  - Path 

# General Purpose SSD (gp2)
  - Baseline
     - 3 IOPS per GiB
     - minimum of 100 IOPS
     - 300 IOPS
  - Burst: up to 3,000 IOPS
