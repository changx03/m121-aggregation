# MongoDB M121 Aggregation

## Chapter 0

```bash
# connection string
mongo "mongodb://cluster0-shard-00-00-jxeqq.mongodb.net:27017,cluster0-shard-00-01-jxeqq.mongodb.net:27017,cluster0-shard-00-02-jxeqq.mongodb.net:27017/aggregations?replicaSet=Cluster0-shard-0" --authenticationDatabase admin --ssl -u m121 -p aggregations --norc
```

```javascript
show collections
```

### The concept of pipelines

- Composition of stages
- Configurable for transformation
- Flow like an assembly line
- Arranged in multiple ways

### Aggregation structure and syntax

```javascript
db.coll.aggregate([{ stage1 }, { stage2 }, { ...stageN }], { options })

// example 1
// $match, $project are aggregate operators
// $in, $gte, $lte, $gt are query operators
db.solarSystem.aggregate(
  [
    {
      $match: {
        atmosphericComposition: { $in: [/O2/] },
        meanTemperature: { $gte: -40, $lte: 40 }
      }
    },
    {
      $project: {
        _id: 0,
        name: 1,
        hasMoons: { $gt: ['$numberOfMoons', 0] }
      }
    }
  ],
  { allowDiskUse: true }
)
// { "name" : "Earth", "hasMoons" : true }
```

[Aggregation Pipeline Quick Reference](https://docs.mongodb.com/manual/meta/aggregation-quick-reference/)

#### Name

- Field path: `$fieldName` (`$numberOfMoons`)
- System variable: `$$UPPERCASE` (`$$CURRENT`)
- User variable: `$$foo`

## Chapter 1 - Basic Aggregation

### Lecture - `$match`

- As early as possible
- Think as a **filter**, rather than `find`
- May contain `$text` query operator, but it must be the first stage in pipeline
- Cannot use `$where` with `$match` (Using `$where` is **risky**! `$where` opens the door for **SQL injection**)
- `$match` uses the same query syntax as `find`

[Reference doc](https://docs.mongodb.com/manual/reference/operator/aggregation/match/)

```js
db.solarSystem
  .aggregate([
    {
      $match: { type: { $ne: 'Star' } }
    }
  ])
  .pretty()

db.solarSystem.find({ type: { $ne: 'Star' } }).pretty()

db.solarSystem.count({ type: { $ne: 'Star' } })
// 8

db.solarSystem.aggregate([
  { $match: { type: { $ne: 'Star' } } },
  { $count: 'planets' }
])
//{ "planets" : 8 }

db.solarSystem.find({ name: 'Earth' }, { _id: 0 })
```

### Lab - `$match`

```bash
mongo "mongodb://cluster0-shard-00-00-jxeqq.mongodb.net:27017,cluster0-shard-00-01-jxeqq.mongodb.net:27017,cluster0-shard-00-02-jxeqq.mongodb.net:27017/aggregations?replicaSet=Cluster0-shard-0" --authenticationDatabase admin --ssl -u m121 -p aggregations --norc
```

```js
show collection
db.movies.findOne()
```

- [x] imdb.rating is at least 7
- [x] genres does not contain "Crime" or "Horror"
- [x] rated is either "PG" or "G"
- [x] languages contains "English" and "Japanese"

```js
var pipeline = [
  {
    $match: {
      $and: [
        { 'imdb.rating': { $gte: 7 } },
        { genres: { $nin: ['Crime', 'Horror'] } },
        { rated: { $in: ['P', 'PG'] } },
        { languages: { $all: ['English', 'Japanese'] } }
      ]
    }
  }
]

// iterator count
db.movies.aggregate(pipeline).itcount()
```

Answer: 15

#### Note

- If performing multiple statement, `$and: [{<query1>}, {<query2>}, ...]` or `$or: [{<query1>}, {<query2>}, ...]` should be used!

### Lecture - `$project`

```js
db.solarSystem.aggregate([{ $project: { <aggregation expression> } }])

// Example
// `_id` field requires explicit exclusion
db.solarSystem.aggregate([{ $project: { _id: 0, name: 1, gravity: 1 } }]) // gravity is an object
db.solarSystem.aggregate([{ $project: { _id: 0, name: 1, 'gravity.value': 1 } }])
// the query above is same as the projection in `find` query

// assign `gravity` field
db.solarSystem.aggregate([{ $project: { _id: 0, name: 1, gravity: '$gravity.value' }}])
// you can rename whatever name you want:
db.solarSystem.aggregate([{ $project: { _id: 0, name: 1, surfaceGravity: '$gravity.value' }}])

// Example: find out how much you weight on different planet
// { $multiply: [ gravityRatio, weightOnEarth ] }
// { $divide: [ "$gravity.value", gravityOfEarth ] }
db.solarSystem.aggregate([{
  $project: {
    _id: 0,
    name: 1,
    myWeight: { $multiply: [ { $divide: [ '$gravity.value', 9.8 ] }, 86 ]}
  }
}])
```

[`$project` doc](https://docs.mongodb.com/manual/reference/operator/aggregation/project/)

### Lab - Changing Document Shape with `$project`

```js
var pipeline = [
  {
    $match: {
      $and: [
        { 'imdb.rating': { $gte: 7 } },
        { genres: { $nin: ['Crime', 'Horror'] } },
        { rated: { $in: ['P', 'PG'] } },
        { languages: { $all: ['English', 'Japanese'] } }
      ]
    }
  },
  {
    $project: {
      _id: 0,
      title: 1,
      rated: 1
    }
  }
]

db.movies.aggregate(pipeline)
```

Answer: 15

### Lab - Computing Fields

#### Question

Find a count of the number of movies that have a title composed of one word

Hint

- [`$split`](https://docs.mongodb.com/manual/meta/aggregation-quick-reference/#string-expressions)
- [`$size`](https://docs.mongodb.com/manual/meta/aggregation-quick-reference/#array-expressions)

```js
var pipeline = [
  {
    $project: {
      title: 1,
      wordCount: { $size: { $split: ['$title', ' '] } }
    }
  },
  {
    $match: {
      wordCount: { $eq: 1 }
    }
  }
]

db.movies.aggregate(pipeline).itcount()
```

Answer: 8068

### Optional Lab - Expressions with \$project

writers is an **array** and it's not **empty**

```js
var pipeline = [
  { $match: { writers: { $elemMatch: { $exists: true } } }
]
db.movies.aggregate(pipeline)

db.movies.findOne({title: "Life Is Beautiful"}, { _id: 0, cast: 1, writers: 1})
```

Problem: "Roberto Benigni" and "Roberto Benigni (story)" both exist.

#### Using `$map`, `$arrayElemAt` and `$split` to extract names

`$$` is referred as `$$this`.

```js
var pipeline = [
  {
    $match: {
      writers: { $elemMatch: { $exists: true } }
    }
  },
  {
    $project: {
      title: 1,
      writers: {
        $map: {
          input: '$writers',
          as: 'writer',
          in: {
            $arrayElemAt: [
              {
                $split: ['$$writer', ' (']
              },
              0
            ]
          }
        }
      }
    }
  }
]

db.movies.aggregate(pipeline)
```

Problem: Find the same person appears in _cast_ and _directors_ and _writers_

```js
db.movies.find({
  cast: { $elemMatch: { $exists: true } },
  directors: { $elemMatch: { $exists: true } },
  writers: { $elemMatch: { $exists: true } }
})

var pipeline = [
  {
    $match: {
      $and: [
        { cast: { $elemMatch: { $exists: true } } },
        { directors: { $elemMatch: { $exists: true } } },
        { writers: { $elemMatch: { $exists: true } } }
      ]
    }
  },
  {
    $project: {
      title: 1,
      cast: 1,
      directors: 1,
      writers: {
        $map: {
          input: '$writers',
          as: 'writer',
          in: {
            $arrayElemAt: [
              {
                $split: ['$$writer', ' (']
              },
              0
            ]
          }
        }
      }
    }
  },
  {
    $project: {
      title: 1,
      commonToAll: {
        $size: { $setIntersection: ['$cast', '$directors', '$writers'] }
      },
      cast: 1,
      directors: 1,
      writers: 1
    }
  },
  {
    $match: {
      commonToAll: { $gte: 1 }
    }
  }
]

db.movies.aggregate(pipeline).itcount()
```

Hint

`$setIntersection` - Takes two or more arrays and returns an array that contains the elements that appear in every input array.

Answer: 1597
