The Ethers Project
==================

ethers.js修改版，支持函数签名替换，也就是有些人需要的没有abi想调用别人函数的功能。

代码需要重新编译ethers.js,请熟练掌握 npm link用法。
[python web3py版在这里](https://github.com/daodao2007/e001)

如需，web3.0客户端定制可以联系我，各种语言均可，提供定制。
智能合约代码分析，也可以联系。


修改代码位置:
packages\contracts\src.ts\index.ts

```
async function populateTransaction(contract: Contract, fragment: FunctionFragment, args: Array<any>): Promise<PopulatedTransaction> {

    // If an extra argument is given, it is overrides
    var fnSignature = null;
    let overrides: CallOverrides = {};
    if (args.length === fragment.inputs.length + 1 && typeof (args[args.length - 1]) === "object") {
        overrides = shallowCopy(args.pop());
    } else if (args.length === fragment.inputs.length + 1 && typeof (args[args.length - 1]) === "string") {
        fnSignature = args.pop();
    }

    // Make sure the parameter count matches
    logger.checkArgumentCount(args.length, fragment.inputs.length, "passed to contract");

    // Populate "from" override (allow promises)
    if (contract.signer) {
        if (overrides.from) {
            // Contracts with a Signer are from the Signer's frame-of-reference;
            // but we allow overriding "from" if it matches the signer
            overrides.from = resolveProperties({
                override: resolveName(contract.signer, overrides.from),
                signer: contract.signer.getAddress()
            }).then(async (check) => {
                if (getAddress(check.signer) !== check.override) {
                    logger.throwError("Contract with a Signer cannot override from", Logger.errors.UNSUPPORTED_OPERATION, {
                        operation: "overrides.from"
                    });
                }
                return check.override;
            });
        } else {
            overrides.from = contract.signer.getAddress();
        }

    } else if (overrides.from) {
        overrides.from = resolveName(contract.provider, overrides.from);

        //} else {
        // Contracts without a signer can override "from", and if
        // unspecified the zero address is used
        //overrides.from = AddressZero;
    }

    // Wait for all dependencies to be resolved (prefer the signer over the provider)
    const resolved = await resolveProperties({
        args: resolveAddresses(contract.signer || contract.provider, args, fragment.inputs),
        address: contract.resolvedAddress,
        overrides: (resolveProperties(overrides) || {})
    });

    // The ABI coded transaction
    var data = null;
    if (fnSignature) {
        data = contract.interface.encodeFunctionDataSign(fragment, resolved.args, fnSignature);
    } else {
        data = contract.interface.encodeFunctionData(fragment, resolved.args);
    }
    const tx: PopulatedTransaction = {
        data: data,
        to: resolved.address
    };

    // Resolved Overrides
    const ro = resolved.overrides;

    // Populate simple overrides
    if (ro.nonce != null) { tx.nonce = BigNumber.from(ro.nonce).toNumber(); }
    if (ro.gasLimit != null) { tx.gasLimit = BigNumber.from(ro.gasLimit); }
    if (ro.gasPrice != null) { tx.gasPrice = BigNumber.from(ro.gasPrice); }
    if (ro.from != null) { tx.from = ro.from; }

    // If there was no "gasLimit" override, but the ABI specifies a default, use it
    if (tx.gasLimit == null && fragment.gas != null) {
        tx.gasLimit = BigNumber.from(fragment.gas).add(21000);
    }

    // Populate "value" override
    if (ro.value) {
        const roValue = BigNumber.from(ro.value);
        if (!roValue.isZero() && !fragment.payable) {
            logger.throwError("non-payable method cannot override value", Logger.errors.UNSUPPORTED_OPERATION, {
                operation: "overrides.value",
                value: overrides.value
            });
        }
        tx.value = roValue;
    }

    // Remvoe the overrides
    delete overrides.nonce;
    delete overrides.gasLimit;
    delete overrides.gasPrice;
    delete overrides.from;
    delete overrides.value;

    // Make sure there are no stray overrides, which may indicate a
    // typo or using an unsupported key.
    const leftovers = Object.keys(overrides).filter((key) => ((<any>overrides)[key] != null));
    if (leftovers.length) {
        logger.throwError(`cannot override ${leftovers.map((l) => JSON.stringify(l)).join(",")}`, Logger.errors.UNSUPPORTED_OPERATION, {
            operation: "overrides",
            overrides: leftovers
        });
    }

    return tx;
}
```
在abi interface内增加encodeFunctionDataSign 函数
修改位置：
packages\abi\src.ts\interface.ts

```
    encodeFunctionDataSign(functionFragment: FunctionFragment | string, values?: Array<any>, fnSign?: string): string {
        if (typeof (functionFragment) === "string") {
            functionFragment = this.getFunction(functionFragment);
        }
        var sighash = null
        if (fnSign != null) {
            sighash = fnSign;
        } else {
            sighash = this.getSighash(functionFragment);
        }

        return hexlify(concat([
            sighash,
            this._encodeParams(functionFragment.inputs, values || [])
        ]));
    }
```
如觉得有点用处请支持[ether.js 无abi调用合约函数，关键代码](https://learnblockchain.cn/goods/33)

开源的已经是全部代码，集市上的在ethers.js 5.0.0的基础上修改，

这个版本在ethers.js master版本上进行。



