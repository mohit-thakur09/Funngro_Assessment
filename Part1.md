1. Race condition while updating the wallet (Critical)

The wallet balance is read first and updated later in a separate query. If two withdrawal requests are processed at the same time, both can read the same balance before either update is committed.

For example, if the user taps the withdraw button twice or the mobile app retries a timed-out request, both requests may succeed and trigger duplicate payouts.

Fix: Use an atomic update (findOneAndUpdate with $inc and a balance condition) or perform the operation inside a MongoDB transaction.

2. Balance check and update are not atomic (Critical)

The balance validation and balance deduction are performed as separate operations. Another request can change the wallet balance between these two steps, making the original validation invalid.

This may allow withdrawals that should have been rejected.

Fix: Perform the balance check and deduction in a single atomic database operation.

3. No protection against duplicate requests (Critical)

The problem states that the mobile app retries timed-out requests, but the endpoint has no idempotency mechanism. A retried request may create multiple payouts for the same withdrawal.

Fix: Use an idempotency key (or unique request ID) and process each withdrawal request only once.

4. Wallet is debited before confirming the payout (Critical)

The wallet balance is reduced before confirming whether the payment gateway has accepted the payout. If the gateway call fails, the user's balance is reduced even though no money is transferred.

This can result in "money debited but not received" issues.

Fix: Record the withdrawal as pending and complete the wallet update using a reliable transaction flow with proper payout confirmation.

5. No database transaction (High)

Updating the wallet and inserting the ledger are separate operations. If the application crashes after updating the wallet but before writing the ledger entry, the system ends up in an inconsistent state.

Fix: Execute all financial operations inside a MongoDB transaction.

6. Ledger does not reflect the actual payout status (High)

A ledger entry is created immediately after calling the payment gateway without checking whether the payout was successful.

This makes reconciliation difficult because the ledger may show a completed withdrawal even though the payment failed.

Fix: Store the transaction with a status such as PENDING, SUCCESS, or FAILED and update it after receiving the gateway response.

7. Missing authorization check (High)

The endpoint accepts userId from the request body. If authentication or ownership validation is not enforced elsewhere, a user could potentially attempt to withdraw money from another user's wallet.

Assumption: Authentication middleware is not shown in the provided code.

Fix: Use the authenticated user's identity from the session or JWT instead of trusting the request body.

8. Missing indexes (Medium)

The question mentions that only the default _id index exists. Queries on Wallets.userId and Ledger.userId will become slower as the collections grow to millions of records.

Fix: Create an index on Wallets.userId and a compound index on Ledger(userId, createdAt) to optimize lookups and transaction history queries.

---------------------------------------------------

// POST /api/wallet/withdraw
// body: { userId, amount, idempotencyKey }

async function withdraw(req, res) {
  const session = await mongoose.startSession();

  try {
    const { userId, amount, idempotencyKey } = req.body;

    // Basic validation
    if (!userId || !amount || amount <= 0 || !idempotencyKey) {
      return res.status(400).json({
        error: "Invalid request",
      });
    }

    session.startTransaction();

    // Prevent duplicate requests
    const existingTxn = await Ledger.findOne(
      { idempotencyKey },
      null,
      { session }
    );

    if (existingTxn) {
      await session.abortTransaction();
      return res.status(409).json({
        error: "Duplicate withdrawal request",
      });
    }

    // Validate user
    const user = await Users.findById(userId).session(session);

    if (!user) {
      await session.abortTransaction();
      return res.status(404).json({
        error: "User not found",
      });
    }

    // Atomically deduct balance
    const wallet = await Wallets.findOneAndUpdate(
      {
        userId,
        balance: { $gte: amount },
      },
      {
        $inc: {
          balance: -amount,
        },
      },
      {
        new: true,
        session,
      }
    );

    if (!wallet) {
      await session.abortTransaction();
      return res.status(400).json({
        error: "Insufficient balance",
      });
    }

    // Create pending ledger entry
    const ledger = await Ledger.create(
      [
        {
          userId,
          type: "withdrawal",
          amount,
          balanceAfter: wallet.balance,
          status: "PENDING",
          idempotencyKey,
          createdAt: new Date(),
        },
      ],
      { session }
    );

    await session.commitTransaction();
    session.endSession();

    // Gateway call should ideally be handled asynchronously
    // using an Outbox/Queue pattern.
    try {
      const payout = await gateway.createPayout({
        account: user.bankAccount,
        amount,
        reference: ledger[0]._id.toString(),
      });

      await Ledger.updateOne(
        { _id: ledger[0]._id },
        {
          $set: {
            status: "SUCCESS",
            gatewayReference: payout.id,
          },
        }
      );

      return res.json({
        ok: true,
        balance: wallet.balance,
      });

    } catch (err) {

      // Compensation logic
      await Wallets.updateOne(
        { userId },
        {
          $inc: {
            balance: amount,
          },
        }
      );

      await Ledger.updateOne(
        { _id: ledger[0]._id },
        {
          $set: {
            status: "FAILED",
            failureReason: err.message,
          },
        }
      );

      return res.status(502).json({
        error: "Payout failed. Amount has been restored.",
      });
    }

  } catch (err) {

    await session.abortTransaction();

    return res.status(500).json({
      error: "Internal server error",
    });

  } finally {

    session.endSession();

  }
}

-----------------------------------

## Before using this system for production-scale payouts, 

- I would separate payout processing from the API using a queue or outbox pattern so that external gateway calls are reliable and retryable. Every withdrawal should have a unique idempotency key to prevent duplicate payouts caused by client retries. 

- I would also maintain explicit transaction states (PENDING, SUCCESS, FAILED) and implement proper monitoring, alerting, and reconciliation jobs to detect and recover failed or delayed payouts. Finally, 

- I would add the required database indexes and comprehensive audit logs to improve performance and support financial compliance.
