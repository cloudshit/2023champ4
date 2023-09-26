# 2023champ4
![image](https://github.com/cloudshit/2023champ4/assets/39158228/34cef44a-2044-4e02-82c8-42aab408c456)
![image](https://github.com/cloudshit/2023champ4/assets/39158228/805e48fb-97b3-4248-939f-dcbc9ebefd13)
![image](https://github.com/cloudshit/2023champ4/assets/39158228/64397a24-0646-4668-a252-47ddc5341ceb)
![image](https://github.com/cloudshit/2023champ4/assets/39158228/6f48e939-61e5-4807-bdac-b51239822930)

## Athena
```SQL
SELECT path, method, statuscode as statusCode, count(*) as count FROM "wsi-glue-database"."athenalog" group by path, method, statuscode;
```

## Redshift
```SQL
SELECT avg(rating)
FROM "dev"."public"."review"
WHERE
    productcategory = 'jean' AND
    authorage BETWEEN 20 AND 29;

SELECT customerid, customerage, customergender
FROM "dev"."public"."order"
ORDER BY productprice DESC
LIMIT 1;
```

## flink (todo)
```sql
%flink.ssql

DROP TABLE wsi_log_test;
CREATE TABLE wsi_log_test (
  `log` STRING
)
WITH (
  'connector' = 'kinesis',
  'stream' = 'log-stream',
  'aws.region' = 'ap-northeast-2',
  'scan.stream.initpos' = 'LATEST',
  'format' = 'json'
);

%flink.ssql
SELECT SPLIT_INDEX(log, ' ', 4) as path, SPLIT_INDEX(log, ' ', 3) as `method`, count(*) as request_count FROM wsi_log_test GROUP BY SPLIT_INDEX(log, ' ', 3), SPLIT_INDEX(log, ' ', 4);
```


## fluentbit
```ini
[SERVICE]
    Parsers_File parsers.conf

[INPUT]
    Name tail
    Path /home/ec2-user/app/app.log

[FILTER]
    Name grep
    Match *
    Exclude log /healthcheck

[OUTPUT]
    Name  kinesis_streams
    Match *
    region ap-northeast-2
    stream log-stream
```

## lambdas.
### order
```js
console.log('Loading function');

exports.handler = async (event, context) => {
    /* Process the list of records and transform them */
    const output = event.records.map((record) => {
        const data = JSON.parse(Buffer.from(record.data, 'base64').toString()).data
        
        
        const result = {
            id: data.id,
            customerid: data.customerID,
            customerage: new Date().getFullYear() - new Date(data.customerBirthday).getFullYear(),
            customergender: data.customerGender === 'm' ? 0 : 1,
            productid: data.productID,
            productcategory: data.productCategory,
            productprice: parseFloat(data.productPrice)
        }
        
        return {
            recordId: record.recordId,
            result: 'Ok',
            data: Buffer.from(JSON.stringify(result)).toString('base64'),
        }
    });
    console.log(`Processing completed.  Successful records ${output.length}.`);
    return { records: output };
};
```

### log
```js
console.log('Loading function');

exports.handler = async (event, context) => {
    /* Process the list of records and transform them */
    const output = event.records.map((record) => {
        const log = JSON.parse(Buffer.from(record.data, 'base64').toString()).log
        
        const columns = log.split(" ")
        const date = new Date(columns[2].replace('(', '').replace(')', ''))
        const path = columns[4]
        const method = columns[3].replace('"', '')
        const statuscode = columns[6]
        const responsetime = columns[7]
    
        
        // ::1 - (2023-09-26T02:10:16Z) "GET / HTTP/1.1 404 0.0 "curl/8.2.1""
        
        
        
        const result = {
            year: date.getFullYear(),
            month: date.getMonth() + 1,
            day: date.getDate(),
            hour: date.getHours(),
            minute: date.getMinutes(),
            second: date.getSeconds(),
            path,
            method,
            statuscode,
            responsetime
        }
        
        return {
            recordId: record.recordId,
            result: 'Ok',
            data: Buffer.from(JSON.stringify(result)).toString('base64'),
        }
    });
    console.log(`Processing completed.  Successful records ${output.length}.`);
    return { records: output };
};
```

### review
```js
console.log('Loading function');

exports.handler = async (event, context) => {
    /* Process the list of records and transform them */
    const output = event.records.map((record) => {
        const data = JSON.parse(Buffer.from(record.data, 'base64').toString()).dynamodb.NewImage
        
        
        const result = {
            id: data.id.S,
            rating: parseFloat(data.rating.S),
            productid: data.product.M.id.S,
            productcategory: data.product.M.category.S,
            authorid: data.author.M.id.S,
            authorage: new Date().getFullYear() - new Date(data.author.M.birthday.S).getFullYear(),
            authorgender: data.author.M.id.S === "m" ? 0 : 1
        }
        
        return {
            recordId: record.recordId,
            result: 'Ok',
            data: Buffer.from(JSON.stringify(result)).toString('base64'),
        }
    });
    console.log(`Processing completed.  Successful records ${output.length}.`);
    return { records: output };
};

```
