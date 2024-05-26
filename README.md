### Optimizing Bulk Payments: A 20x Improvement with PL/SQL Stored Procedures

In the realm of financial services, processing bulk cash and holding payments efficiently is crucial for maintaining smooth operations and ensuring timely transactions. Our recent project aimed to optimize the processing time for these bulk payments, and through the use of PL/SQL stored procedures, we achieved a remarkable 20x reduction in processing times, adding substantial value to our operational workflows. This article provides an overview of the challenges we faced, the solutions we implemented, and the results of our optimization efforts.

#### The Problem

Our legacy implementation for processing bulk cash holding payments was primarily handled through application code, which resulted in several performance issues:

- **High Population of Data**: The system had to process a large volume of transactions.
- **Multiple Direct Database Calls**: Each operation involved numerous direct calls to the database, adding significant latency.
- **Extensive Process**: The process covered various stages, from transaction initiation to notification sending.

These factors combined led to substantial delays, making the system inefficient and slow.

#### The Solution

To address these issues, we implemented a hybrid approach, combining application code with stored procedures. By moving the heavy-lifting database operations into the database itself, we significantly reduced the latency and improved the overall performance.

###Add component utilization graphic 

#### Implementation

**1. Code Level Changes**

We introduced PL/SQL procedures to handle the bulk of the processing. Here's an overview of the key changes made at the code level:

- **Initial Bulk Process Start**:
    ```java
    logger.info("LN:xxx", "Bulk Payment Request processing started through DB : sp_bulk_paying_process_c ");
    getOmsTransactionManageCountryWrapperLocal().validatePayingBulkStatus(payingBulkRequestBean.getOmsMsgHeader().getTenantCode(), payingBulkRequestBean, 1);
    getOmsTransactionManageCountryWrapperLocal().validatePayingBulkStatus(payingBulkRequestBean.getOmsMsgHeader().getTenantCode(), payingBulkRequestBean, 2);
    ```

- **Validation Method**:
    ```java
    public void validatePayingAgentBulkStatus(String tenantCode, PayingBulkRequestBean payingBulkRequestBean, int processType) {
        try {
            getPaymentDetailDAO().updateBulkPayingProcessStatus(bulkPayingRequestBean, processType);
        } catch (Exception ex) {
            logger.error("Exception While Updating paying Agent status Update, RollBack Call " + ex);
            getSessionContext().setRollbackOnly();
        }
    }
    ```

- **Database Update Method**:
    ```java
    public boolean updateBulkPayingProcessStatus(BulkPayingRequestBean bulkPayingRequestBean, int processType) {
        Connection connection = null;
        CallableStatement cs = null;
        try {
            connection = getDBConnectionXA(bulkPayingRequestBean.getOmsMsgHeader().getTenantCode());
            cs = connection.prepareCall("{call sp_bulk_paying_process_c(?,?,?,?)}");
            cs.setLong(OMSConst.NUMBER_1, payingAgentRequestBean.getRequestBody().getPayingSessionId());
            cs.setInt(OMSConst.NUMBER_2, processType);
            cs.setInt(OMSConst.NUMBER_3, payingAgentRequestBean.getOmsMsgHeader().getLoginID());
            cs.setInt(OMSConst.NUMBER_4, payingAgentRequestBean.getRequestBody().getInstituteId());
            cs.executeQuery();
        } catch (Exception e) {
            logger.error("LN:116", "Error executing call sp_bulk_paying_process_c", e);
            throw new OMSCoreDAOException(ErrorCodes.DAO_ERROR, e);
        } finally {
            close(connection, cs);
        }
        return true;
    }
    ```

**2. Database Level Changes**

The core processing logic was moved into a PL/SQL procedure. Here’s a simplified version of the `sp_paying_agent_process_c` procedure:

```sql
PROCEDURE sp_bulk_paying_process_c (
    p_t500_id          IN NUMBER,
    p_type             IN NUMBER,  -- 1 = Initial bulk process, 2 = process
    p_user_id          IN NUMBER,
    p_institution_id   IN NUMBER
) IS
BEGIN
    IF (p_type = 1) THEN
        UPDATE dfn_ntp.u06_cash_account
           SET u06_bulk_process_status_c = 1
         WHERE u06_id IN (SELECT u07.u07_cash_account_id_u06
                          FROM t501_payment_detail_c t501
                          JOIN u07_trading_account u07 ON u07.u07_exchange_account_no = t501.t501_account_code
                          WHERE t501_status_id_v01 = 2 AND t501_payment_session_id_t500 = p_t500_id);
    ELSIF (p_type = 2) THEN
        -- Detailed processing logic here
        -- At the end, we set u06_bulk_process_status_c = 2 
    END IF;
END;
```

#### Results

By moving the data-intensive operations to the database, we achieved:

- **10x Faster Processing**: Significant reduction in processing time due to minimized latency.
- **Reduced Latency**: Fewer direct database calls from the application layer.
- **Scalability**: Improved scalability and efficiency of the system, capable of handling larger volumes of transactions seamlessly.

#### Conclusion

Optimizing bulk cash holding adjustments by leveraging PL/SQL procedures has proven to be highly effective. This approach not only enhances performance but also simplifies the codebase by delegating complex operations to the database. The success of this project underscores the importance of database optimization in improving system performance and scalability.

By continually refining and optimizing our processes, we can ensure that our systems remain robust, efficient, and capable of meeting the demands of modern financial operations.
