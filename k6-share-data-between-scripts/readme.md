# Problem:

## When using k6 to run performance tests, we wanted to split the script into two tests: One performing the POSTs and one performing the GETs. 

The problem is that k6's test life-cycle doesn't allow passing data outside of the default test loop. I needed to collect the unique orderIDs
created during the POST test script and pass those to the GET test script.

The hacky solution? It comes in two parts.

Initialize an array in the INIT stage of the POST test script. During the TESTLOOP, capture the orderIDs and populate the array.

Here comes the tricky part. The array will still be empty if you try to reference it in the TEARDOWN. Data flows one way in k6, from INIT to other stages only.
SETUP cannot pass data to TESTLOOP. TESTLOOP cannot pass data to TEARDOWN. 

So, we get tricky back. Knowing how many iterations/loops the test will run, we put a conditional statement at the bottom of the TESTLOOP.

```
if (`${__ITER}` == (options.iterations -1).toString()) {
  let dataTransfer = JSON.parse('{"orders": []}');
  dataTransfer['orders'] = orderIdArray;
  console.log(JSON.stringify(dataTransfer));
}
```

Basically, create a JSON object and store the array of orderIDs in the object. Then print the object as a string to console.

The second part of the magic trick is that we invoke k6 through the command line. 
```
k6 run --logformat=raw --out orderIDs.json post-test-script.js
```
That command will strip all the timestamp/debug info from the console out and create a JSON extension file with a JSON formatted string in it containing the desired orderIDs.

From there, we just import that JSON file into the GET test scripts during the INIT phase and all is well in the world.
