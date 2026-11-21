/*============================================================
  Customer AR Aging Summary Report

  Purpose:
  - Summarizes outstanding AR balances by customer.
  - Buckets balances into aging ranges as of the report date.
  - Includes:
      1. Open invoices
      2. Unapplied / On-account receipts
      3. Unapplied credit memos
=============================================================*/

SELECT
    TRUNC(CAST(:P_AS_OF_DATE AS DATE) + 1/4) AS as_of_date,

    /* Customer dimensions */
    xep.name            AS legal_entity_name,
    cust.account_number,
    p.party_name        AS account_name,
    prof.credit_limit,

    /*--------------------------------------------------------
      Aging buckets
      Balance = Invoice Amount + Adjustments - Applications
    --------------------------------------------------------*/
    SUM(
        CASE
            WHEN TO_NUMBER(
                     TRUNC(CAST(:P_AS_OF_DATE AS DATE) + 1/4)
                     - TRUNC(inv.due_date)
                 ) <= 0
            THEN inv.amount_due_original
                 + NVL(adj.adjusted_amount, 0)
                 - NVL(app.applied_amount, 0)
        END
    ) AS balance_current,

    SUM(
        CASE
            WHEN TO_NUMBER(
                     TRUNC(CAST(:P_AS_OF_DATE AS DATE) + 1/4)
                     - TRUNC(inv.due_date)
                 ) BETWEEN 1 AND 30
            THEN inv.amount_due_original
                 + NVL(adj.adjusted_amount, 0)
                 - NVL(app.applied_amount, 0)
        END
    ) AS balance_1_to_30,

    SUM(
        CASE
            WHEN TO_NUMBER(
                     TRUNC(CAST(:P_AS_OF_DATE AS DATE) + 1/4)
                     - TRUNC(inv.due_date)
                 ) BETWEEN 31 AND 60
            THEN inv.amount_due_original
                 + NVL(adj.adjusted_amount, 0)
                 - NVL(app.applied_amount, 0)
        END
    ) AS balance_31_to_60,

    SUM(
        CASE
            WHEN TO_NUMBER(
                     TRUNC(CAST(:P_AS_OF_DATE AS DATE) + 1/4)
                     - TRUNC(inv.due_date)
                 ) BETWEEN 61 AND 90
            THEN inv.amount_due_original
                 + NVL(adj.adjusted_amount, 0)
                 - NVL(app.applied_amount, 0)
        END
    ) AS balance_61_to_90,

    SUM(
        CASE
            WHEN TO_NUMBER(
                     TRUNC(CAST(:P_AS_OF_DATE AS DATE) + 1/4)
                     - TRUNC(inv.due_date)
                 ) BETWEEN 91 AND 120
            THEN inv.amount_due_original
                 + NVL(adj.adjusted_amount, 0)
                 - NVL(app.applied_amount, 0)
        END
    ) AS balance_91_to_120,

    SUM(
        CASE
            WHEN TO_NUMBER(
                     TRUNC(CAST(:P_AS_OF_DATE AS DATE) + 1/4)
                     - TRUNC(inv.due_date)
                 ) BETWEEN 121 AND 150
            THEN inv.amount_due_original
                 + NVL(adj.adjusted_amount, 0)
                 - NVL(app.applied_amount, 0)
        END
    ) AS balance_121_to_150,

    SUM(
        CASE
            WHEN TO_NUMBER(
                     TRUNC(CAST(:P_AS_OF_DATE AS DATE) + 1/4)
                     - TRUNC(inv.due_date)
                 ) > 150
            THEN inv.amount_due_original
                 + NVL(adj.adjusted_amount, 0)
                 - NVL(app.applied_amount, 0)
        END
    ) AS balance_gt_150,

    /* Collections reporting bucket */
    SUM(
        CASE
            WHEN TO_NUMBER(
                     TRUNC(CAST(:P_AS_OF_DATE AS DATE) + 1/4)
                     - TRUNC(inv.due_date)
                 ) > 90
            THEN inv.amount_due_original
                 + NVL(adj.adjusted_amount, 0)
                 - NVL(app.applied_amount, 0)
        END
    ) AS balance_gt_90,

    /* Total customer exposure */
    SUM(
        inv.amount_due_original
        + NVL(adj.adjusted_amount, 0)
        - NVL(app.applied_amount, 0)
    ) AS total_balance

FROM hz_cust_accounts cust,
     hz_parties p,
     hz_customer_profiles_f prof,
     ar_payment_schedules_all inv,
     ra_customer_trx_all rct,

     /*--------------------------------------------------------
       Retrieve and concatenate BOL numbers.
       Limited to 4 values for display purposes.
     --------------------------------------------------------*/
     (
         SELECT
             customer_trx_id,
             LISTAGG(
                 ila.interface_line_attribute1,
                 ','
             ) WITHIN GROUP (ORDER BY ila.rn) AS bol_numbers
         FROM (
             SELECT
                 customer_trx_id,
                 CASE
                     ROW_NUMBER() OVER (
                         PARTITION BY customer_trx_id
                         ORDER BY interface_line_attribute1
                     )
                     WHEN 4 THEN '...'
                     ELSE interface_line_attribute1
                 END AS interface_line_attribute1,
                 ROW_NUMBER() OVER (
                     PARTITION BY customer_trx_id
                     ORDER BY interface_line_attribute1
                 ) rn
             FROM ra_customer_trx_lines_all
             WHERE interface_line_attribute1 IS NOT NULL
             GROUP BY
                 customer_trx_id,
                 interface_line_attribute1
         ) ila
         WHERE ila.rn <= 4
         GROUP BY ila.customer_trx_id
     ) rctl,

     ra_cust_trx_types_all rtt,
     xle_entity_profiles xep,

     /*--------------------------------------------------------
       Approved adjustments posted before report date
     --------------------------------------------------------*/
     (
         SELECT
             aps.payment_schedule_id,
             SUM(NVL(adj.acctd_amount, 0)) adjusted_amount
         FROM ar_payment_schedules_all aps,
              ar_adjustments_all adj
         WHERE aps.customer_trx_id = adj.customer_trx_id
           AND aps.org_id = adj.org_id
           AND aps.class = 'INV'
           AND adj.status = 'A'
           AND TRUNC(adj.gl_date)
               <= TRUNC(CAST(:P_AS_OF_DATE AS DATE) + 1/4)
         GROUP BY aps.payment_schedule_id
     ) adj,

     /*--------------------------------------------------------
       Applied receipts and credit memos as of report date
     --------------------------------------------------------*/
     (
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
         GROUP BY ara.applied_payment_schedule_id
     ) app

WHERE inv.class = 'INV'

    /* Customer joins */
    AND inv.customer_id = cust.cust_account_id
    AND cust.party_id = p.party_id
    AND cust.cust_account_id = prof.cust_account_id
    AND prof.site_use_id IS NULL

    /* Optional transaction attributes */
    AND inv.customer_trx_id = rct.customer_trx_id(+)
    AND rct.customer_trx_id = rctl.customer_trx_id(+)
    AND rct.legal_entity_id = xep.legal_entity_id(+)
    AND inv.cust_trx_type_seq_id = rtt.cust_trx_type_seq_id(+)

    /* Adjustments and applications */
    AND inv.payment_schedule_id = adj.payment_schedule_id(+)
    AND inv.payment_schedule_id = app.applied_payment_schedule_id(+)

    /* Report filters */
    AND TRUNC(inv.gl_date)
        <= TRUNC(CAST(:P_AS_OF_DATE AS DATE) + 1/4)

    AND xep.name IN (:P_LEGAL_ENTITY_NAME)

    AND cust.attribute2 =
        NVL(:P_CUSTOMER_TYPE, cust.attribute2)

    /* Ignore fully settled invoices */
    AND inv.amount_due_original
      + NVL(adj.adjusted_amount, 0)
      - NVL(app.applied_amount, 0) <> 0

GROUP BY
    xep.name,
    cust.account_number,
    p.party_name,
    prof.credit_limit

/*============================================================
  Unapplied and On-Account Receipts
  Added as negative balances since they reduce AR exposure.
=============================================================*/
UNION ALL

SELECT
    TRUNC(CAST(:P_AS_OF_DATE AS DATE) + 1/4),
    NULL,
    cust.account_number,
    p.party_name,
    prof.credit_limit,

    -SUM(NVL(ara.amount_applied, 0)) AS balance_current,

    0, 0, 0, 0, 0, 0, 0,

    -SUM(NVL(ara.amount_applied, 0)) AS total_balance

FROM ar_receivable_applications_all ara
INNER JOIN ar_payment_schedules_all aps
    ON ara.payment_schedule_id = aps.payment_schedule_id
INNER JOIN hz_cust_accounts cust
    ON aps.customer_id = cust.cust_account_id
INNER JOIN hz_parties p
    ON cust.party_id = p.party_id
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
    prof.credit_limit

HAVING SUM(NVL(ara.amount_applied, 0)) <> 0

/*============================================================
  Unapplied Credit Memos
=============================================================*/
UNION ALL

SELECT
    TRUNC(CAST(:P_AS_OF_DATE AS DATE) + 1/4),
    NULL,
    cust.account_number,
    p.party_name,
    prof.credit_limit,

    SUM(NVL(cm.amount_due_original, 0)) AS balance_current,

    0, 0, 0, 0, 0, 0, 0,

    SUM(NVL(cm.amount_due_original, 0)) AS total_balance

FROM ar_payment_schedules_all cm

INNER JOIN hz_cust_accounts cust
    ON cm.customer_id = cust.cust_account_id

INNER JOIN hz_parties p
    ON cust.party_id = p.party_id

INNER JOIN hz_customer_profiles_f prof
    ON cust.cust_account_id = prof.cust_account_id
   AND prof.site_use_id IS NULL

LEFT JOIN ar_receivable_applications_all ara
    ON cm.payment_schedule_id = ara.payment_schedule_id

WHERE cm.class = 'CM'
  AND cust.attribute2 = NVL(:P_CUSTOMER_TYPE, cust.attribute2)
  AND TRUNC(cm.gl_date)
      <= TRUNC(CAST(:P_AS_OF_DATE AS DATE) + 1/4)

  /* Credit memo not yet applied */
  AND ara.payment_schedule_id IS NULL

GROUP BY
    cust.account_number,
    p.party_name,
    prof.credit_limit

HAVING SUM(NVL(cm.amount_due_original, 0)) <> 0;
