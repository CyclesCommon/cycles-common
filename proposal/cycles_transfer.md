# Cycles Common Transfer Interface

## Objective

To establish a standard interface for canisters to transfer cycles.
Each actor may choose to implement the interface in slightly different ways, but as long as they conform to a common interface, they can make calls to each other.

## Interface

The proposed interface consists of only one function, described below in Candid.

```
type Response =
 variant {
   rejected: text;
   transferred: nat64;
   unauthorized;
 };
type Request =
 record {
   amount: opt nat64;
   receiver: opt principal;
   sender: opt principal;
 };
type CyclesTransfer =
 service {
   cycles_transfer: (Request) -> (Response);
 };
```

Canisters that wish to support cycles transfer should implement a `cycles_transfer` function that takes a transfer `Request` and gives a transfer `Response`.
This function can be used to either send or receive cycles, or even serve as intermediaries that transfer cycles on behalf of other users or canisters.

**Request type**

The function input type `Request` has several fields that are all marked as optional.
Their intended meaning is explained below, but it will depend on an actual implementation of this interface.

* `amount:` The amount of cycles that should be transferred. If unspecified, it defaults to the amount of cycles that is sent along this call.
* `receiver:` Who should receive the cycles. If unspecified, it defaults to the target canister of this call.
* `sender:` Who initiated this transfer. If unspecified, it defaults to the caller.

As a simple example, if none of the fields are given, the `cycles_transfer` function can treat the request as depositing all available cycles.

**Response type**

The function return type `Response` lists 3 possible cases.

* `transferred`: the actual amount of cycles accepted by the receiver.
* `unauthorized`: the caller is not authorized to initiate the transfer.
* `rejected`: the transfer is rejected for reasons other than being `unauthorized`.

An implementation may also choose to throw an error instead of returning one of the responses.

Upon either a success or an error return, part or all cycles that were sent along with the call may have already been consumed (accepted) by the callee.
The caller can use the `ic0.msg_cycles_refunded` system call to check if that is the case.

**WARNING**

An important principle when working with this interface is that
***One must not make any assumption or implicitly trust either the caller or the callee of the `cycles_transfer` function:***

* The callee of `cycles_transfer` (i.e. its implementation) should not blindly trust values from the input `Request`.
* The caller of `cycles_transfer` should not blindly trust values returned from `Response`.
* It is recommended to only call `cycles_transfer` on canisters that either you own or your can trust.

For a public service implementing this interface, if they want to be trusted they should:
* publish the source code, and
* offer ways for users to verify the installed canister binary actually matches the published source code.

## Use cases

The reference code of the above proposed interface is provided as [a Motoko module in the *cycles-motoko* repo](https://github.com/CyclesCommon/cycles-motoko/blob/main/src/Cycles.mo) that can be included as part of a Motoko project.
A rust implementation will be provided later.

Examples in this section are also written in Motoko.
All code can be found in the [examples sub-directory](https://github.com/CyclesCommon/cycles-motoko/tree/main/examples) in the *cycles-motoko* repo.

### Collect remaining cycles before uninstalling a canister

Before we uninstall a canister for good, we may want to collect its remaining cycles that otherwise may be lost forever.

The following code shows how this transfer-out can be implemented.
We can re-install an existing canister with this code, and then call the `transfer_cycles` method with a designated receiver (another canister that implements this interface, please see the next example).

```
import Cycles "../src/Cycles";
import Option "mo:base/Option";
import Principal "mo:base/Principal";

// An actor class is required if we want to use shared(msg) at initialization time.
shared(msg) actor class TransferOut() {
  let OWNER = msg.caller; // The creator of this canister becomes the owner.
  let MINIMUM_RESERVE = 400_000_000_000 : Nat64; // Minimum required to finish the call, subject to change.

  // Allow transferring of all remaining cycles (less a minimum reserve required to finish
  // the call) if the following conditions are satisified:
  //  - the call is made by the owner;
  //  - the receiver field is not null.
  public shared(msg) func cycles_transfer(request: Cycles.Request) : async Cycles.Response {
     if (msg.caller == OWNER) {
       assert(request.sender == null);
       assert(request.amount == null);
       assert(request.receiver != null);
       let receiver_id = Option.unwrap(request.receiver);
       let receiver : Cycles.Transfer = actor(Principal.toText(receiver_id));
       let balance = Cycles.balance();
       assert(balance >= MINIMUM_RESERVE);
       Cycles.add(balance - MINIMUM_RESERVE);
       await receiver.cycles_transfer({ amount = null; sender = null; receiver = null })
     } else {
       #unauthorized
     }
  };

  // Return the principal of the creator of this canister
  public query func owner() : async Principal {
    return OWNER;
  }
}
```

### Receive cycles sent from other canisters

The above example requires a canister to receive the cycles.
This canister can be written as follows:

```
import Cycles "../src/Cycles";
import Option "mo:base/Option";
import Principal "mo:base/Principal";

actor TransferIn {
  // Accept incoming cycles transfer when the receiver field is null.
  public func cycles_transfer(request: Cycles.Request) : async Cycles.Response {
     if (request.receiver == null) {
       // Deposit all available cycles to this canister.
       let available = Cycles.available();
       let requested_amount = Option.get(request.amount, available);
       let amount = if (requested_amount > available) { available } else { requested_amount };
       let accepted = Cycles.accept(amount);
       return #transferred(accepted);
     } else {
       #unauthorized
     }
  }
}
```

### Both send and receive

We can combine both send and receive functionalities in a single actor by merging the above code.
While we are at it, we can also allow the user to specify the amount of cycles to transfer.

[The full source code](https://github.com/CyclesCommon/cycles-motoko/blob/main/examples/Transfer.mo) is provided in the [cycles-motoko] library.

### Forward cycles on behalf of the caller

One may also choose to implement the `cycles_transfer` interface as a forwarding service, i.e., a canister that delivers cycles to the designated receiver on behalf of the caller.
There can be many reasons why one wants to do this: to implement access control, to allow redirection, or to circumvent the limitations imposed by the network (e.g. at the moment one cannot send cycles from an "ordinary" application subnet to a "verified" one).

The actual implementation is left as an exercise to the readers.

[vessel]: https://github.com/dfinity/vessel
[cycles-motoko]: https://github.com/CyclesCommon/cycles-motoko
