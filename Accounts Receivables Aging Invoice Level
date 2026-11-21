/*=========================================================
  AR Aging Report
  Purpose:
  - Returns open invoices, unapplied/on-account receipts,
    and unapplied credit memos as of the selected date.
  - Calculates aging buckets and customer balances.
=========================================================*/

SELECT
    /* As-of date with timezone offset handling */
    TRUNC(CAST(:P_AS_OF_DATE AS DATE) + 1/4) AS as_of_date,

    inv.payment_schedule_id,

    /* Customer & legal entity details */
    xep.name                                 AS legal_entity_name,
    cust.account_number,
    p.party_name                             AS account_name,
    prof.credit_limit,

    /* Invoice details */
    inv.trx_number                           AS invoice_number,
    rct.purchase_order                       AS customer_po_number,
    rctl.bol_numbers,
    rtt.name                                 AS transaction_type,

    TRUNC(inv.trx_date)                      AS transaction_date,
    TRUNC(inv.due_date)                      AS due_date,

    /* Number of days past due */
    TO_NUMBER(
        TRUNC(CAST(:P_AS_OF_DATE AS DATE) + 1/4)
        - TRUNC(inv.due_date)
    ) AS past_due_days,

    /* Original invoice balance */
    inv.amount_due_original                  AS original_balance,

    /* Adjustment and application totals */
    adj.adjusted_amount,
    app.received_amount,
    app.credited_amount,
    app.applied_amount,

    /*-----------------------------------------------------
      Aging Buckets
      Balance = Original Amount
              + Approved Adjustments
              - Applied Payments/Credits
    -----------------------------------------------------*/

    -- Current (not yet due)
    CASE
        WHEN TO_NUMBER(
                 TRUNC(CAST(:P_AS_OF_DATE AS DATE) + 1/4)
                 - TRUNC(inv.due_date)
             ) <= 0
        THEN inv.amount_due_original
             + NVL(adj.adjusted_amount, 0)
             - NVL(app.applied_amount, 0)
    END AS balance_current,

    -- 1-30 days overdue
    CASE
        WHEN TO_NUMBER(
                 TRUNC(CAST(:P_AS_OF_DATE AS DATE) + 1/4)
                 - TRUNC(inv.due_date)
             ) BETWEEN 1 AND 30
        THEN inv.amount_due_original
             + NVL(adj.adjusted_amount, 0)
             - NVL(app.applied_amount, 0)
    END AS balance_1_to_30,

    -- 31-60 days overdue
    CASE
        WHEN TO_NUMBER(
                 TRUNC(CAST(:P_AS_OF_DATE AS DATE) + 1/4)
                 - TRUNC(inv.due_date)
             ) BETWEEN 31 AND 60
        THEN inv.amount_due_original
             + NVL(adj.adjusted_amount, 0)
             - NVL(app.applied_amount, 0)
    END AS balance_31_to_60,

    -- 61-90 days overdue
    CASE
        WHEN TO_NUMBER(
                 TRUNC(CAST(:P_AS_OF_DATE AS DATE) + 1/4)
                 - TRUNC(inv.due_date)
             ) BETWEEN 61 AND 90
        THEN inv.amount_due_original
             + NVL(adj.adjusted_amount, 0)
             - NVL(app.applied_amount, 0)
    END AS balance_61_to_90,

    -- 91-120 days overdue
    CASE
        WHEN TO_NUMBER(
                 TRUNC(CAST(:P_AS_OF_DATE AS DATE) + 1/4)
                 - TRUNC(inv.due_date)
             ) BETWEEN 91 AND 120
        THEN inv.amount_due_original
             + NVL(adj.adjusted_amount, 0)
             - NVL(app.applied_amount, 0)
    END AS balance_91_to_120,

    -- 121-150 days overdue
    CASE
        WHEN TO_NUMBER(
                 TRUNC(CAST(:P_AS_OF_DATE AS DATE) + 1/4)
                 - TRUNC(inv.due_date)
             ) BETWEEN 121 AND 150
        THEN inv.amount_due_original
             + NVL(adj.adjusted_amount, 0)
             - NVL(app.applied_amount, 0)
    END AS balance_121_to_150,

    -- Greater than 150 days overdue
    CASE
        WHEN TO_NUMBER(
                 TRUNC(CAST(:P_AS_OF_DATE AS DATE) + 1/4)
                 - TRUNC(inv.due_date)
             ) > 150
        THEN inv.amount_due_original
             + NVL(adj.adjusted_amount, 0)
             - NVL(app.applied_amount, 0)
    END AS balance_gt_150,

    /* Special collection reporting bucket */
    CASE
        WHEN TO_NUMBER(
                 TRUNC(CAST(:P_AS_OF_DATE AS DATE) + 1/4)
                 - TRUNC(inv.due_date)
             ) > 90
        THEN inv.amount_due_original
             + NVL(adj.adjusted_amount, 0)
             - NVL(app.applied_amount, 0)
    END AS balance_gt_90,

    /* Outstanding invoice balance */
    inv.amount_due_original
        + NVL(adj.adjusted_amount, 0)
        - NVL(app.applied_amount, 0)         AS total_balance

FROM hz_cust_accounts cust

    INNER JOIN hz_parties p
        ON p.party_id = cust.party_id

    INNER JOIN hz_customer_profiles_f prof
        ON cust.cust_account_id = prof.cust_account_id
       AND prof.site_use_id IS NULL

    /* Invoice schedules */
    INNER JOIN ar_payment_schedules_all inv
        ON cust.cust_account_id = inv.customer_id

    LEFT JOIN ra_customer_trx_all rct
        ON inv.customer_trx_id = rct.customer_trx_id

    LEFT JOIN xle_entity_profiles xep
        ON rct.legal_entity_id = xep.legal_entity_id

    LEFT JOIN ra_cust_trx_types_all rtt
        ON inv.cust_trx_type_seq_id = rtt.cust_trx_type_seq_id

    /*-----------------------------------------------------
      BOL Numbers
      Show max 4 values and append "..." if more exist
    -----------------------------------------------------*/
    LEFT JOIN (
        SELECT
            customer_trx_id,
            LISTAGG(ila.interface_line_attribute1, ',')
                WITHIN GROUP (ORDER BY ila.rn) AS bol_numbers
        FROM (
            SELECT
                customer_trx_id,
                CASE
                    WHEN ROW_NUMBER() OVER (
                             PARTITION BY customer_trx_id
                             ORDER BY interface_line_attribute1
                         ) = 4
                    THEN '...'
                    ELSE interface_line_attribute1
                END AS interface_line_attribute1,
                ROW_NUMBER() OVER (
                    PARTITION BY customer_trx_id
                    ORDER BY interface_line_attribute1
                ) rn
            FROM ra_customer_trx_lines_all
            WHERE interface_line_attribute1 IS NOT NULL
            GROUP BY customer_trx_id,
                     interface_line_attribute1
        ) ila
        WHERE ila.rn <= 4
        GROUP BY ila.customer_trx_id
    ) rctl
        ON rct.customer_trx_id = rctl.customer_trx_id

    /*-----------------------------------------------------
      Approved invoice adjustments applied
      before the report as-of date
    -----------------------------------------------------*/
    LEFT JOIN (
        SELECT
            aps.payment_schedule_id,
            SUM(NVL(adj.acctd_amount, 0)) adjusted_amount
        FROM ar_payment_schedules_all aps
             JOIN ar_adjustments_all adj
               ON aps.customer_trx_id = adj.customer_trx_id
              AND aps.org_id = adj.org_id
        WHERE aps.class = 'INV'
          AND adj.status = 'A'
          AND TRUNC(adj.gl_date)
                <= TRUNC(CAST(:P_AS_OF_DATE AS DATE) + 1/4)
        GROUP BY aps.payment_schedule_id
    ) adj
        ON inv.payment_schedule_id = adj.payment_schedule_id

    /*-----------------------------------------------------
      Cash receipts and credit memo applications
      posted before the as-of date
    -----------------------------------------------------*/
    LEFT JOIN (
        SELECT
            ara.applied_payment_schedule_id,

            SUM(
                CASE
                    WHEN ara.application_type = 'CASH'
                    THEN NVL(ara.amount_applied, 0)
                END
            ) AS received_amount,

            SUM(
                CASE
                    WHEN ara.application_type = 'CM'
                    THEN NVL(ara.amount_applied, 0)
                END
            ) AS credited_amount,

            SUM(NVL(ara.amount_applied, 0)) AS applied_amount

        FROM ar_receivable_applications_all ara
        WHERE ara.application_type IN ('CASH', 'CM')
          AND ara.status = 'APP'
          AND TRUNC(ara.gl_date)
                <= TRUNC(CAST(:P_AS_OF_DATE AS DATE) + 1/4)

          -- Excluded reversal logic retained for future use
          -- AND NVL(ara.reversal_gl_date,
          --         DATE '9999-12-31')
          --         > TRUNC(CAST(:P_AS_OF_DATE AS DATE) + 1/4)

        GROUP BY ara.applied_payment_schedule_id
    ) app
        ON inv.payment_schedule_id = app.applied_payment_schedule_id

WHERE inv.class = 'INV'
  AND cust.attribute2 = NVL(:P_CUSTOMER_TYPE, cust.attribute2)
  AND TRUNC(inv.gl_date)
        <= TRUNC(CAST(:P_AS_OF_DATE AS DATE) + 1/4)
  AND xep.name IN (:P_LEGAL_ENTITY_NAME)

  /* Exclude fully settled invoices */
  AND inv.amount_due_original
        + NVL(adj.adjusted_amount, 0)
        - NVL(app.applied_amount, 0) <> 0

  /* Open invoices only */
  AND inv.status = 'OP'

/*=========================================================
  UNAPPLIED / ON-ACCOUNT CASH RECEIPTS
=========================================================*/
UNION ALL

SELECT
    TRUNC(CAST(:P_AS_OF_DATE AS DATE) + 1/4) AS as_of_date,
    -1                                       AS payment_schedule_id,
    NULL                                     AS legal_entity_name,
    cust.account_number,
    p.party_name                             AS account_name,
    prof.credit_limit,

    CASE ara.status
        WHEN 'UNAPP' THEN 'Unapplied'
        WHEN 'ACC'   THEN 'On-account'
    END AS invoice_number,

    NULL,
    NULL,
    NULL,
    NULL,
    NULL,

    0 AS past_due_days,
    0 AS original_balance,
    0 AS adjusted_amount,
    0 AS received_amount,
    0 AS credited_amount,

    SUM(NVL(ara.amount_applied, 0)) AS applied_amount,

    /* Negative because receipt reduces AR exposure */
    -SUM(NVL(ara.amount_applied, 0)) AS balance_current,

    0,0,0,0,0,0,0,

    -SUM(NVL(ara.amount_applied, 0)) AS total_balance

FROM ar_receivable_applications_all ara
INNER JOIN ar_payment_schedules_all aps
    ON ara.payment_schedule_id = aps.payment_schedule_id
INNER JOIN hz_cust_accounts cust
    ON aps.customer_id = cust.cust_account_id
INNER JOIN hz_parties p
    ON p.party_id = cust.party_id
INNER JOIN hz_customer_profiles_f prof
    ON cust.cust_account_id = prof.cust_account_id
   AND prof.site_use_id IS NULL

WHERE ara.application_type IN ('CASH', 'CM')
  AND ara.status IN ('UNAPP', 'ACC')
  AND cust.attribute2 = NVL(:P_CUSTOMER_TYPE, cust.attribute2)
  AND TRUNC(ara.gl_date)
        <= TRUNC(CAST(:P_AS_OF_DATE AS DATE) + 1/4)

GROUP BY
    cust.account_number,
    p.party_name,
    prof.credit_limit,
    ara.status

HAVING SUM(NVL(ara.amount_applied, 0)) <> 0

/*=========================================================
  UNAPPLIED CREDIT MEMOS
=========================================================*/
UNION ALL

SELECT
    TRUNC(CAST(:P_AS_OF_DATE AS DATE) + 1/4) AS as_of_date,
    -1                                       AS payment_schedule_id,

    xep.name                                 AS legal_entity_name,
    cust.account_number,
    p.party_name                             AS account_name,
    prof.credit_limit,

    cm.trx_number                            AS invoice_number,

    NULL,
    NULL,

    rtt.name                                 AS transaction_type,
    cm.trx_date                              AS transaction_date,

    NULL AS due_date,
    0    AS past_due_days,

    SUM(NVL(cm.amount_due_original, 0))      AS original_balance,

    0 AS adjusted_amount,
    0 AS received_amount,
    0 AS credited_amount,
    0 AS applied_amount,

    SUM(NVL(cm.amount_due_original, 0))      AS balance_current,

    0,0,0,0,0,0,0,

    SUM(NVL(cm.amount_due_original, 0))      AS total_balance

FROM ar_payment_schedules_all cm

INNER JOIN hz_cust_accounts cust
    ON cm.customer_id = cust.cust_account_id

INNER JOIN hz_parties p
    ON p.party_id = cust.party_id

INNER JOIN hz_customer_profiles_f prof
    ON cust.cust_account_id = prof.cust_account_id
   AND prof.site_use_id IS NULL

LEFT JOIN ra_customer_trx_all rct
    ON cm.customer_trx_id = rct.customer_trx_id

LEFT JOIN xle_entity_profiles xep
    ON rct.legal_entity_id = xep.legal_entity_id

LEFT JOIN ar_receivable_applications_all ara
    ON cm.payment_schedule_id = ara.payment_schedule_id

LEFT JOIN ra_cust_trx_types_all rtt
    ON cm.cust_trx_type_seq_id = rtt.cust_trx_type_seq_id

WHERE cm.class = 'CM'
  AND cust.attribute2 = NVL(:P_CUSTOMER_TYPE, cust.attribute2)
  AND TRUNC(cm.gl_date)
        <= TRUNC(CAST(:P_AS_OF_DATE AS DATE) + 1/4)
  AND ara.applied_payment_schedule_id IS NULL
  AND xep.name IN (:P_LEGAL_ENTITY_NAME)
  AND cm.acctd_amount_due_remaining <> 0

GROUP BY
    cust.account_number,
    p.party_name,
    prof.credit_limit,
    cm.trx_number,
    cm.trx_date,
    rtt.name,
    xep.name

HAVING SUM(NVL(cm.amount_due_original, 0)) <> 0;
