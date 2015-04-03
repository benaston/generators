

Generators are a two-way communication between a 
generator (a value-generator, if you like) and a consumer.

The following code demonstrates this.

    'use strict';
    
    function* go() {
      console.log(yield 'a'); // We are the generator.
      console.log(yield 'b');
      return 'c';
    }
    
    var g = go();
    console.log(g.next()); // We are the consumer.
    console.log(g.next('one'));
    console.log(g.next('two')); 


    function *go() {
      yield 'whatever';
    }
    var g = go();
    (function() {
      console.log(g.next()); 
      console.log(g.next()); 
    }());
    
    var v = ['a', 'b'];
    
    function* go() {
      console.log(yield v.shift());
    }
    
    var g = go();
    g.next('ba');
    
    function* echo(text, delay) {
        delay = delay == null ? 0 : delay;
        const caller = yield;
        setTimeout(function() {
            caller.success(text);
        }, delay);
    }
    
    document.getElementById('go').onclick = function() {
        run(function* echoes() {
            console.log(yield echo('this')); // return generator object
            console.log(yield echo('is'));
            console.log(yield echo('a test'));
        });
    };
    
    // Needed to avoid the caller having to 
    // invoke the generator themselves.
    function run(genFunc) {
        runGenObj(genFunc());
    }
    
    // Run the generator object `genObj`, 
    // report results via the callbacks in `callbacks`.
    function runGenObj(genObj, callbacks) {
        callbacks = callbacks || undefined;
        handleOneNext();
    
        /**
         * Handle one invocation of `next()`:
         * If there was a `prevResult`, it becomes the parameter.
         * What `next()` returns is what we have to run next.
         * The `success` callback triggers another round,
         * with the result assigned to `prevResult`.
         */
        function handleOneNext(prevResult) {
            try {
                prevResult = prevResult || null;
                let yielded = genObj.next(prevResult); // may throw
                if (yielded.done) {
                    if (yielded.value !== undefined) {
                        // Something was explicitly returned:
                        // Report the value as a result to the caller
                        callbacks.success(yielded.value);
                    }
                } else {
                    setTimeout(runYieldedValue, 0, yielded.value);
                }
            }
            // Catch unforeseen errors in genObj
            catch (error) {
                if (callbacks) {
                    callbacks.failure(error);
                } else {
                    throw error;
                }
            }
        }
    
        function runYieldedValue(yieldedValue) {
            if (yieldedValue === undefined) {
                // If code yields `undefined`, it wants callbacks
                handleOneNext(callbacks);
            } else if (Array.isArray(yieldedValue)) {
                runInParallel(yieldedValue);
            } else {
                // Yielded value is a generator object
                runGenObj(yieldedValue, {
                    success(result) {
                            handleOneNext(result);
                        },
                        failure(err) {
                            genObj.throw(err);
                        },
                });
            }
        }
    
        function runInParallel(genObjs) {
            let resultArray = new Array(genObjs.length);
            let resultCountdown = genObjs.length;
            for (let e of genObjs.entries()) {
                let i = e.i;
                let genObj = e.genObj;
                runGenObj(genObj, {
                    success(result) {
                            resultArray[i] = result;
                            resultCountdown--;
                            if (resultCountdown <= 0) {
                                handleOneNext(resultArray);
                            }
                        },
                        failure(err) {
                            genObj.throw(err);
                        },
                });
            }
        }
}

