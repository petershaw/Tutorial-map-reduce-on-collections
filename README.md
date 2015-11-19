# JavaaScript Tutorial : map reduce on collections - avoid nested iterrations like foreach

This is a possible solution for a module that solves a problem from a real life 
implementation my company have today. The code is perfect to show some functional 
programming styles, that is small enough to show in a blog post.  

## Download 

You can download the code and play with it. If you have some improvements please share it
with me. 

Go to the download folder and run 

```bash
npm install
npm start
```

You'll neede node >= 4 to run this tutorial. 
All Anotations are in the [Sourcefile](source.js), too. 

## Description

One ordering system (A) should commit the order to a vendor system (B). A is a modern 
scripting framework, fully RESTIfyed. Mobile Apps talk to this Service to place and pay
orers. B is an old fashion 16-Bit single-core system-on-a-chipm, like a softdrink 
automat.
 
The automat doesn't understand JSON or XML. It wants to get the order in an ugly 
but powerfull custom format. To understand the implementation, you have to understand 
the machine first. 
The softdrink automat has 16 Slots with different drinks in it. 


 16 | 15 | 14 | 13 | 12 | 11 | 10 | 9  | 8  | 7  | 6  | 5  | 4  | 3  | 2  | 1
----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----
 C  | S  | M  | W  | F  | B  | R  | B  | G  | T  | O  | F  |    |    |    |    
 O  | P  | A  | A  | A  | I  | E  | E  | I  | O  | R  | R  |    |    |    |    
 K  | R  | T  | T  | N  | O  | D  | E  | N  | N  | A  | I  |    |    |    |    
 E  | I  | E  | E  | T  | N  |    | R  | G  | I  | N  | T  |    |    |    |    
    | T  |    | R  | A  | A  | W  |    | E  | C  | G  | Z  |    |    |    |    
    | E  |    |    |    | D  | I  |    | R  |    | I  |    |    |    |    |    
    |    |    |    |    | E  | N  |    |    |    | N  | C  |    |    |    |     
    |    |    |    |    |    | E  |    | A  |    | A  | O  |    |    |    |      
    |    |    |    |    |    |    |    | L  |    |    | L  |    |    |    |       
    |    |    |    |    |    |    |    | E  |    |    | A  |    |    |    |       
    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |       


To proceed an order it have two data fields to submit from the app to the machine:
   * validOrder
   * priceOrder

validOrder is a 16-Bit STRING-value that indicated the slot of the machine. For 
every bought product the corresponding bit must be set to high. 
For example: to get a "Mate" out of this maschine validOrder must be: `0000000000001000`.
To get a Mate and a cold Beer validOrder must be set to: `0000000010001000`.

priceOrder must be set correctly, too. The priceOrder is an array-like STRING of cents 
onto the right array index.
Let's say a Coke costs $ 2.50. That the first position (from behind) must be 250. For
our example with the Mate ($ 2.50) and the cold Beer ($ 2.80) the priceOrder array 
should be: `0,0,0,0,0,0,0,0,280,0,0,0,250,0,0,0`

We can not look inside the maschine, but we belive that the input will be pared in c
to int and pushed to a shift-register to turn on the motors of the slots. 

*More than one product*:
Ok, let's say a friend comes in and wants a beer, too. The validOrder is still 
`0000000010001000`, but the price has to be increased 
to: `0,0,0,0,0,0,0,0,560,0,0,0,250,0,0,0`
because 560 is 2 * 280, thats the cent of a beer for $ 2.80. The validOrder must be 
the same, because it triggers only the active slots. 

So far so good, but our modern ordering system does not know anything about slot
positions. Hm, we are dealing with so called PLU (price look-up code)
Luckily we can set the PLU like we want, so we define a table of plus that are the 
power of two (https://en.wikipedia.org/wiki/Power_of_two)
 
    Coke:            1
    Sprite:          2
    Mate:            4 
    Water:           8
    Fanta:          16
    Bionade:        32
    Red Wine:       64
    Beer:          128
    Ginger Ale:    256
    Tonic:         512
    Orangina:     1024
    Fritz Cola:   2048

## Different Thought 

Well my first thought in my functional programming world was map/reduce the plu's to 
it's corresponding bit values, while my college was more on the iterating side of this
town. 
Well nothing is more or less correct until you do one way right. There is no one 
final solution to this problem. There are many ways. I present one way in this post,
a way that i do like the most. That does not ment that this is the best one. 


## Implementation

I implement the functional way here to demonstrate the approach of functional units. 
Feel free to do a nice and clever way of your own and share it with me. I like to 
see other solutions and I really like to read code. 
 
I will annotate the source to demonstrate and teach the approach. Feel free to comment 
the steps. It is mastering time, folks! 

The code is written from top to bottom. Maybe it is not like i would commit into the 
corporate SCM, but it will show how I think while coding and more how functional 
programming in JavaScript supports me.

## The Code

### We use `let` and other stuff from modern ECMA script

```Javascript
"use strict";
```
 
## Modules we use

```Javascript
let _ = require('underscore')
    ;
```    

### PRE-REQUIREMENTS
a order is placed, we habe a array of PLU's
1 x Code
2 X Beer

```Javascript
let order = [
      {plu: 1,   price: 150}
    , {plu: 128, price: 280}
    , {plu: 128, price: 280}
    ];
```

In this example we habe a 16-Bit vendor machine

```Javascript
const BITS = 16
```


### First thoughts about how we can solve the porblem.

We habe two separate outputs. The calculation of `valid` is diffrent to 
the calculation to `price`. So I will seerate them from each other. That will be good
for testing and debugging, too. 
I wrap them in functions first, befor I implement the rest. A good style is to do the 
underlaying mechanism first. 
My function design is to call each of it by a single PLU.

I have to implement two functions 

  * getValidationStringForPLU 
  * getPriceArrayStringForItem
  

### Transform a PLU into its binary

Parameter: PLU (the product PLU)
Parameter: padding (a length of the returned string)

Usage: `getValidationStringForPLU(plu, BITS);`
The result for 128 is '0000000010000000'

```JavaScript
let getValidationStringForPLU = function getValidationStringForPLU(plu, len){
    /** 
     * Because the plu is in the power of two, we just need to transform it into the 
     * binary representation
     */
    var binaryString = (parseInt(plu) >>> 0).toString(2);

    // if a len is set, than pad the string to the right.
    if(len){
        binaryString = '0'.repeat(       // repeat to add a '0' 
            len - binaryString.length    // for the times that is the differences between
                                         // the current length and given len
        ) + binaryString;                // and than append the original string
    }
    return binaryString;
};
```
  
### Transform a item with price into its price-array-position

Parameter: Item (the product object)
Parameter: size (the size of the returned array)

The result for 128 is '[ , , , , , , , , 280, , , , , , ,    ]'

```JavaScript
let getPriceArrayStringForItem = function getPriceArrayStringForItem(item, size){
    var price = new Array(size);
    /**
     * The position in the array is the product from: 
     * BASETONUM(LOG(PLU, 2), 10)
     * But: Position 0 is the last, and position 16 is the first place in the array, so
     * turn it around  with ABS(current - size).
     */
     let position = Math.abs(
        Math.log2(parseInt(item.plu))
        - (size -1)
     );

     price[position] = item.price;       // set the price to the korrekt position
     return price;
};
```


###  will map every plu to its own price array.

```
    PLU - - - \
                 [,,,,,,,,280,,,,,,,]
    PLU - - - -  [,,,,,,,,280,,,,,,,] ---reduce---> [,,,,,,,,560,,,,,,,250]
                 [,,,,,,,,,,,,,,, 50]
    PLU - - - /
```
   
This can be done asynchron and does not need any other requirements. You can do it
on parallel or on different machines. This is the MAPPING part. Later on I will show 
the REDUCEING part of the m/r, because I need something to reduce the arrays into 
a single one.
But how will the reducing work? It will need some 'processor' that knows what to do.
The smalest piece i can imagine is (well i can image a photon or a hicks, but not 
in real life - and that is real life code...) to sum up two arrays. 
More specific what i want to have later on is a function that adds every element in
an array to the same position of another array.

`sumPrice` will add a price array to another. That is my smalest unit - the precesor of 
the reduce. 

I could use a function like: 
`let sumPrice = function sumPrice(pricesLeft, pricesRight){ .. }`
..., but I prefere Prototypes for simple tasks on Objects. We have an array, and want  
to add another array to it, so why don't extend the Array.Object with this
functionality. 

```JavaScript
Array.prototype.sumArray = function(arr) {                      // Prototyping Array
    var sum = [];                                               // a Temporary new Array 
    if (arr != null && this.length == arr.length) {             // check validation
        for (var i = 0; i < arr.length; i++) {                  // itterate every element
            /**
             * Add the element to the temporary array 
             * with the product of both elements or 0
             */
            sum.push((parseInt(this[i]) || 0) + (parseInt(arr[i]) || 0));
            /**
             * !!! ATTENTION
             * Yes, I could override the elements in 'this', instead of returning the new 
             * array. But would it be more readable? I expect a returning result from a 
             * function call.
             * Maybe I should rewrite it that it behaves exactly like pop() and push() 
             * later on. 
             */            
        }
    } else { console.error("missmatch array summarising", this, arr); } // loging
    return sum; // return the temporary array as a new result 
};
```

### Now I have all I need! I can implement the calculation part

```JavaScript
 // first I get the valid-string of all items, therefore I have to prepare the collection
let uniqueProductPlus = _.unique(       // each plu MUST be quniqe
    _.map(order, function(item){        // map the orders to get only the PLU
        return item.plu;
    })
);
```


### GET VALID ORDER
Because every plu is now unique and in the power of two it is possible to sum it 
p. And get the binary string at once. The calculation is easy:
Coke 1  =   00000001 
Beer 128 =  00010000
so a Coke and a Beer is 1 + 128 = 129. And 129 in binary is :
            00010001

```JavaScript
let validOrder = getValidationStringForPLU(
    // count all PLUs together
    _.reduce(uniqueProductPlus, function(n, memo){ return n + memo; }, 0)
    // padding to 16
    , BITS
);
console.log( "Valid Order:", validOrder );  // <- Eh, thats the whole trick. ;-)
```

ok fine, whats next. 50% done. Now, I need the price array...     
     

### GET VALID PRICE

I should slightly do the same, but on all PLUs. First I'll map them all into 
price arrays like described above, ...

```JavaScript
let priceArrays = _.map(order, function(item){
    return getPriceArrayStringForItem(item, BITS);
});
```


...than I reduce it to one array with my processor function prototype. 
Here it sounds good, not to tweak the array itself and return a new array. So I 
decided not to refactor the code above.

```JavaScript
let validPrice = _.reduce(priceArrays, function(n, memo){   // reduce all arrays
    return memo.sumArray(n);                                // by the processor sumArray
                                                            // with the last result.
}, new Array(BITS));                                        // up from an empty array.
console.log( "Valid Price:", validPrice );  // <- Year! Print it out
```


## CONSOLE OUTPUT

```bash
    Valid Order: 0000000010000001
    Valid Price: [ 0, 0, 0, 0, 0, 0, 0, 0, 560, 0, 0, 0, 0, 0, 0, 150 ]
```
 
## Postscriptum 
I hope this brings a bit more fun to functional programming without side effects.
node.js and swift are really good language for data-crunching tasks like this. It is
so much better for to "think in a language". 
Remember: divide a Problem into single units and than implement from the unit up to 
the solution. 

    If you like this tutorial and want to say thanks, than please spend a fraction of a 
    BitCoin to: 1DtvkCh28zqarTEUHtxs7gWtutsv2Cnf9d

Cheers, petershaw.
kris@ausdertechnik.de

