# Trading Automation System

---

## **Table of Contents**

[1. **System Overview**](#1-system-overview)
   - [1.1 System Goals](#system-goals)
   - [1.2 Trading System Components](#trading-system-components)

[2. **Architecture**](#2-architecture)
   - [2.1 IBridge](#1-ibridge)
   - [2.2 Trading Recommendation System (TRS)](#2-trading-recommendation-system-trs)
   - [2.3 Trading System Manager (TSM)](#3-tsm-trading-system-manager)
   - [2.4 User Interface (Web App)](#31-user-interface-web-app)
   - [2.5 Files Specification and Diagrams](#architecture-and-specification)
     - [2.5.1 Specification of Trading Instruction Files](#specification-of-trading-instruction-files-json)
     - [2.5.2 Specification of General Log Files](#specification-of-general-log-files-raw-text)
     - [2.5.3 Specification of Trades Log Files](#specification-of-trades-log-files-json)
     - [2.5.4 Trading System Architecture Diagram](#trading-system-architecture-diagram)
     - [2.5.5 Trading System Data Flow Diagram](#trading-system-data-flow-diagram)

[3. **Features and Usage**](#3-features-and-usage)
   - [3.1 Features](#21-features)
     - [3.1.1 IBridge Features](#ibridge-features)
     - [3.1.2 Trading Recommendation System (TRS) Features](#trading-recommendation-system-trs-features)
     - [3.1.3 Trading System Manager (TSM) Features](#trading-system-manager-tsm-features)
     - [3.1.4 User Interface (Web App) Features](#user-interface-web-app-features)
   - [3.2 Usage Scenarios](#22-usage-scenarios)
     - [3.2.1 Executing Trades via TSM](#221-executing-trades-via-tsm)
     - [3.2.2 Monitoring PnL and Trade Status](#222-monitoring-pnl-and-trade-status)
     - [3.2.3 Managing and Debugging Errors](#223-managing-and-debugging-errors)

[4. **Proposed Timeline**](#4-proposed-timeline)
   - [4.1 Milestone Summary](#milestone-summary)

---

## **1. System Overview**

The purpose of this project is to build a robust and automated **Trading System Management (TSM)**. This system executes trading instructions provided by an external **Trading Recommendation System (TRS)**, handling real-time trade execution, monitoring, and logging. 

### **System Goals:**
- **Automation**: Minimize manual intervention in trading processes
- **Real-Time Monitoring**: Track and manage the trade lifecycle in real-time
- **Reliability**: Execute trading recommendations reliably
- **Robustness**: Ensure the system operates effectively with minimal downtime

### **Trading System Components:**

1. **IBridge**: Handles data requests and manages communication between TSM, TRS, and Interactive Brokers (IB).
2. **Trading Recommendation System (TRS)**: Provides daily asset trading recommendations.
3. **Trading Manager Server (TSM)**: Executes and monitors trades, and communicates with the user interface (UI) for real-time updates.

---

## **2. Architecture**:

### 1. **IBridge**:
- **Goals**: 
  - Ensure persistent connectivity with Interactive Brokers for reliable trade execution.
  - Accurately download and maintain daily trade data and logs for further analysis.
- **Instance Type**: EC2 instance (on-demand).
- **Functionality**: 
  - Maintains a persistent IB login session.
  - Downloads daily historical and live market data.
  - Executes trading instructions from TSM.
  - Retrieves updates on executed trades (including PnL, errors).
- **Communication**: 
  - Secures traffic with inbound access only from TRS and TSM using security groups.
  - Facilitates communication between IB, TRS, and TSM for smooth trade management.

### 2. **Trading Recommendation System (TRS)**:
- **Goals**: 
  - Provide clear and timely daily recommendations for asset trading on the Russell 1000.
  - Recommendations should balance profit and risk.
- **Instance Type**: EC2 instance (on-demand).
- **Functionality**:
  - Downloads daily data (market info, past trades) and preprocesses it.
  - Generates daily trading recommendations using the core prediction model.
  - Uploads recommendations to S3 for access by TSM.
- **Communication**: 
  - Interacts with IBridge via HTTP API for data access.
  - Uploads recommendations to S3 for TSM to retrieve and execute.

### 3. **TSM (Trading System Manager)**:
- **Goals**: 
  - Ensure reliable execution of trading recommendations from TRS.
  - Monitor trade execution and system performance in real-time.
  - Provide consistent updates on portfolio performance (PnL).
- **Instance Type**: EC2 instance, scheduled to run during trading hours.
- **Functionality**:
  - Downloads daily recommendations from S3.
  - Processes trading instructions and converts recommendations into executable trades.
  - Tracks asset performance during trading hours, managing entry points.
  - Monitors the progress and status of executed trades.
  - Retrieves PnL and trade errors for analysis.
  - Sends trade updates and logs to the User Interface for real-time tracking.
- **Communication**:
  - Interacts with IBridge for executing and managing trades.
  - Retrieves TRS predictions from S3 for trade execution.

### 3.1 **User Interface (Web App)**:
- **Goals**: 
  - Provide a clear, real-time dashboard for monitoring trade status, PnL, and system logs.
  - Facilitate easy tracking of the daily trading process for users.
- **Functionality**:
  - Displays all daily recommendations.
  - Tracks the progress of trades, including status updates and real-time PnL.
  - Provides a log of trade executions and errors for troubleshooting.
- **Communication**: 
  - Securely interacts with the Trading Manager via HTTP API to display live trade updates and portfolio information.
  - Security is based on an internal private network.

---

### Architecture and Specification

#### **Specification of Trading Instruction Files (JSON)**

The trading instruction file is a JSON file generated after the system processes recommendations and strategy rules. This file is used to execute trades via IBridge.

##### **Example Format**:
```json
{
  "AAPL": {
    "_comment": "OCA: one-cancel-all",
    "Entry OCA": [
      {
        "On": "cross-over"
        "Action": "BUY",
        "Order": "limit",
        "Qty": 1,
        "price": "69.484375"
      },
      {
        "On": "cross-under"
        "Action": "SELL",
        "Order": "limit",
        "Qty": 1,
        "price": "70.58981"
      }
    ],
    "Orders if Long": [
      {
        "_comment": "Exit on Touch",
        "Action": "SELL",
        "Order": "limit",
        "Qty": 1,
        "price": "70.58981"
      },
      {
        "_comment": "SL - 4-diff",
        "Action": "SELL",
        "Order": "limit",
        "Qty": 1,
        "price": "65.04"
      },
      {
        "_comment": "Buy another position - DCA",
        "Action": "BUY",
        "Order": "Market",
        "Qty": 1,
        "price": "67.28"
      }
    ],
    "Orders if Short": [
      {
        "_comment": "Exit on Touch",
        "Action": "BUY",
        "Order": "limit",
        "Qty": 1,
        "price": "69.484375"
      },
      {
        "_comment": "SL - 4-diff",
        "Action": "SELL",
        "Order": "limit",
        "Qty": 1,
        "price": "74.98"
      },
      {
        "_comment": "Sell another position - DCA",
        "Action": "BUY",
        "Order": "Market",
        "Qty": 1,
        "price": "72.78"
      }
    ]
  }
}
```

##### **Field Descriptions**:
- **Asset Symbol (e.g., AAPL)**: Represents the stock symbol for which the trades are being executed.
- **Entry OCA on Cross**: Defines the One-Cancels-All (OCA) orders on the crossover condition.
  - **Action**: BUY/SELL action.
  - **Order**: The type of order (limit, market, etc.).
  - **Qty**: Quantity to be traded.
  - **Price**: The price at which the order is executed.
- **Orders if Long/Short**: Contains the set of actions to be executed if the system is in a long or short position.
  - Includes exit conditions, stop-loss (SL), and dollar-cost averaging (DCA) strategies.

---

#### **Specification of General Log Files (Raw Text)**

General log files are raw text files that capture system-wide activities, including errors, warnings, and critical system events. These logs are crucial for debugging and auditing the system.

##### **Example Format**:
```
[2024-09-24 10:30:45] INFO: Starting TSM server.
[2024-09-24 10:31:00] INFO: Fetching recommendations from S3.
[2024-09-24 10:31:15] INFO: Processing recommendation for AAPL: Buy at 69.484375, Sell at 70.58981.
[2024-09-24 10:32:00] WARNING: Delay in receiving response from IBridge.
[2024-09-24 10:32:30] ERROR: Failed to execute trade for AAPL: Insufficient margin.
[2024-09-24 10:33:00] INFO: Trade execution complete for AAPL. PnL: +3.5%.
```

##### **Field Descriptions**:
- **Timestamp**: Date and time when the log entry was generated.
- **Log Level**: The severity level of the log message (INFO, WARNING, ERROR).
- **Message**: A description of the event, including actions being performed, errors encountered, or completion statuses.

---

#### **Specification of Trades Log Files (JSON)**

The trades log file tracks each executed trade and its outcome. It contains details about the executed order, price, quantity, and any associated PnL. This file is typically stored in JSON format.

##### **Example Format**:
```json
{
  "timestamp": "2024-09-24T10:33:00Z",
  "symbol": "AAPL",
  "trade_id": "trade_123456",
  "action": "BUY",
  "order_type": "limit",
  "qty": 1,
  "executed_price": 69.484375,
  "status": "executed",
  "PnL": 3.5,
  "comments": "Exit triggered at upper boundary."
}
```

##### **Field Descriptions**:
- **Timestamp**: The time when the trade was executed.
- **Symbol**: The asset symbol (e.g., AAPL).
- **Trade ID**: A unique identifier for the trade.
- **Action**: BUY or SELL action.
- **Order Type**: The type of order (limit, market, etc.).
- **Qty**: The number of shares/contracts traded.
- **Executed Price**: The price at which the trade was executed.
- **Status**: The status of the trade (executed, pending, failed).
- **PnL**: The profit or loss associated with the trade (if available).
- **Comments**: Any additional information or comments about the trade.

---

#### **Trading System Architecture Diagram**

![Trading system Arch Flow Diagram](/Diagrams/TradingSystem.jpg)

#### **Trading System Data Flow Diagram**

![Trading system Data Flow Diagram](/Diagrams/DFD-trading-system.jpg)
---

## **3. Features and Usage**

This section will dive into the specific features of each component of the Trading System, detailing how it works and how users can interact with the system. Each subsection will describe the unique capabilities and how the system can be effectively utilized in real-world scenarios.

### **2.1 Features**

#### **IBridge Features**:
- **Persistent IB Connection**: Ensures the system remains logged in to Interactive Brokers, maintaining a continuous trading session.
- **Automated Trade Execution**: Receives and executes trades based on the orders processed by TSM.
- **Historical and Live Data Retrieval**: Downloads daily market data for trade analysis and monitoring.
- **Trade Status and Error Reporting**: Tracks the success or failure of each trade, logging errors for debugging.

#### **Trading Recommendation System (TRS) Features**:
- **Automated Data Collection and Preprocessing**: Downloads, cleans, and prepares daily data for model input.
- **Prediction Generation**: Runs deep learning models to generate actionable trading recommendations.
- **Seamless S3 Integration**: Uploads prediction results directly to S3 for easy access by TSM.

#### **Trading System Manager (TSM) Features**:
- **Trade Processing**: Downloads recommendations from S3 and processes them into actionable trades.
- **Real-Time Trade Monitoring**: Continuously tracks the state of each trade, monitoring entry points, and managing execution status.
- **PnL Calculation**: Provides up-to-the-minute profit and loss calculations for active trades.
- **Error Handling**: Captures and logs any issues encountered during the trading process for easier resolution.

#### **User Interface (Web App) Features**:
- **Daily Recommendations View**: Displays all recommended trades for the day.
- **Trade Status Updates**: Real-time updates on trade progress, including execution status and any errors encountered.
- **PnL Dashboard**: Displays the performance of the portfolio, providing insights into profits and losses.
- **System Logs**: Allows users to view logs for debugging and trade validation.

---

### **2.2 Usage Scenarios**

#### **2.2.1 Executing Trades via TSM**:
- **Step 1**: The system starts the TSM during trading hours, which then pulls daily trading recommendations from the S3 bucket.  
- **Step 2**: TSM processes these recommendations, converting them into trades.
- **Step 3**: The trades are sent to IBridge, which logs into the IB session and executes the trades.
- **Step 4**: Real-time trade updates are sent back to TSM, which monitors and logs them.
- **Step 5**: The User Interface provides live updates on trade execution status and PnL throughout the day.


#### **2.2.2 Monitoring PnL and Trade Status**:
- **Step 1**: Users log into the Web App to view daily recommendations.
- **Step 2**: The User Interface displays real-time information on trade execution, including order status and error logs.
- **Step 3**: Users monitor PnL, seeing how the trades are performing over time.
  - **Trade Status** displayed in the UI:
    
  | Asset Symbol | Action | Status   | Executed Price | PnL   |  
  |--------------|--------|----------|----------------|-------|  
  | AAPL         | Buy    | Executed | 150.25         | +3.50%|  
  | MSFT         | Sell   | Pending  | --             | --    |
  
#### **2.2.3 Managing and Debugging Errors**:
- **Step 1**: If any trades fail or errors occur, they are captured and logged in real-time.
- **Step 2**: Users can view the error logs via the User Interface, identifying potential issues with trade execution.
- **Step 3**: Debugging steps can be taken based on the detailed logs, which can be passed to developers for resolution.
  
---

## **4. Proposed Timeline**

**Total Estimated Timeline: 12-16 weeks (3-4 months)**

This timeline allows time for development, thorough testing, and integration to ensure the entire system functions as expected, with proper handling of trades, monitoring, and error management.

| **Task**                                    | **Description**                                                                 | **Estimated Time**       | **Key Activities**                                                                                  | **Milestone**                                      |
|---------------------------------------------|---------------------------------------------------------------------------------|--------------------------|------------------------------------------------------------------------------------------------------|---------------------------------------------------|
| **Initial Setup**                           | AWS environment setup (EC2, S3, IAM, security groups, networking)               | 1-2 weeks                | Setup EC2 instances, S3 buckets, IAM roles, and configure security groups for secure communication   |                 |
| **IBridge Development**                     | Build IBridge for IB API integration, trade execution, and data retrieval       | 3-4 weeks                | IB API integration, trade execution, real-time updates, error handling, persistent session management|           |
| **IBridge Testing**                         | Test IBridge with demo accounts and live data retrieval                         | 1 week                   | Test trade execution, error handling, and logging                                                     | **Milestone 1: IBridge Completed**                  |
| **TRS (Trading Recommendation System) Development** | Develop TRS for recommendation generation and S3 uploads                         | 2-3 weeks                | Data preprocessing, recommendation engine, S3 integration                                              |                   |
| **TRS Testing**                             | Validate recommendation generation and S3 uploads                               | 1 week                   | Test recommendations with sample data and ensure S3 file accuracy                                     | **Milestone 2: TRS Completed**                     |
| **TSM (Trading System Manager) Development** | Build TSM for trade processing, risk management, and real-time trade tracking   | 3-4 weeks                | Process S3 recommendations, manage trade execution, PnL tracking, error handling                     |              |
| **TSM Testing**                             | Test trade processing, risk management, and PnL tracking                        | 1-2 weeks                | Simulate trade recommendations, test strategies, ensure accurate trade tracking                      | **Milestone 3: TSM Core Completed**                     |
| **User Interface (Web App) Development**    | Build UI for displaying trade statuses, PnL, and logs                           | 2-3 weeks                | Design and develop real-time dashboard for trade tracking, PnL display, and system logs              |              |
| **User Interface Testing**                  | Front-end testing for proper display of trade statuses and logs                 | 1 week                   | Ensure trade status, PnL, and logs are correctly displayed in the UI                                  | **Milestone 4: UI Core Completed**                      |
| **Component Integration**                   | Combine IBridge, TRS, TSM, and UI into a single system                          | 2-3 weeks                | End-to-end testing of data flow, trade execution, real-time monitoring, and error handling            | **Milestone 5: Components Integrated**          |
| **System Testing**                          | Stress testing and edge case handling for end-to-end functionality              | 1 week                   | Test system under load, handle trade errors, ensure system scalability                                |             |
| **Final Deployment**                        | Deploy the full system on AWS for production                                    | 1 week                   | Deploy all components, final configurations, monitor system in production                             | **Milestone 6: Full System Deployed**                |

---

### **Milestone Summary**:

1. **Milestone 1: IBridge Completed** – IBridge development finished.
2. **Milestone 2: TRS Completed** - Trading Recommendation System completed.
3. **Milestone 3: TSM Core Completed** – Trading Manager Server developed.
4. **Milestone 4: UI Core Completed** -  User Interface dashboard developed.
5. **Milestone 5: Components Integrated** - All components integrated for seamless operation.
6.**Milestone 6: Full System Deployed** - End-to-end system tested, ready for production, deployed and live on AWS.