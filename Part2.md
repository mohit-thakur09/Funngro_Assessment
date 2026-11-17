# MongoDB collections with fields and indexes.

## Users

{
  _id: ObjectId,
  name: String,
  email: String,
  cohort: String,
  createdAt: Date
}

----------

## Wallets

{
  _id: ObjectId,
  userId: ObjectId,
  balance: Number,
  updatedAt: Date
}

----------

## RewardLedger

{
  _id: ObjectId,
  userId: ObjectId,
  type: String,          // EARN | CONVERT | EXPIRE
  source: String,        // PROJECT | PROMOTION
  points: Number,
  conversionRate: Number,
  walletAmount: Number,
  status: String,        // PENDING | SUCCESS | FAILED
  createdAt: Date
}


------------

## ConversionRates

{
  _id: ObjectId,
  rate: Number,
  effectiveFrom: Date,
  effectiveTo: Date
}

---------------------------------------------------------------

## Aggregation Pipeline

db.RewardLedger.aggregate([
  {
    $match: {
      createdAt: {
        $gte: startDate,
        $lt: endDate
      }
    }
  },
  {
    $group: {
      _id: "$userId",
      earned: {
        $sum: {
          $cond: [{ $eq: ["$type", "EARN"] }, "$points", 0]
        }
      },
      converted: {
        $sum: {
          $cond: [{ $eq: ["$type", "CONVERT"] }, "$points", 0]
        }
      },
      expired: {
        $sum: {
          $cond: [{ $eq: ["$type", "EXPIRE"] }, "$points", 0]
        }
      }
    }
  }
]);

----------------------------------------------

## Preventing Double Conversion

Point conversion should be handled inside a MongoDB transaction using an atomic update. Before processing a request, the system should verify that the reward points are still available and create a ledger entry with a unique transaction (or idempotency) key. If the same request is received again, the existing transaction is returned instead of processing it again. This ensures that the same reward points cannot be converted multiple times, even under concurrent requests.

