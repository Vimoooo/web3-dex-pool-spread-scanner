import os
from web3 import Web3

RPC_URL = os.getenv("RPC_URL")
FACTORY_ADDRESSES = os.getenv("FACTORY_ADDRESSES", "").split(",")

TOKEN_A = os.getenv("TOKEN_A")
TOKEN_B = os.getenv("TOKEN_B")

FACTORY_ABI = [
    {
        "type": "function",
        "name": "getPair",
        "stateMutability": "view",
        "inputs": [
            {"name": "tokenA", "type": "address"},
            {"name": "tokenB", "type": "address"},
        ],
        "outputs": [{"name": "pair", "type": "address"}],
    }
]

PAIR_ABI = [
    {
        "type": "function",
        "name": "getReserves",
        "stateMutability": "view",
        "inputs": [],
        "outputs": [
            {"name": "reserve0", "type": "uint112"},
            {"name": "reserve1", "type": "uint112"},
            {"name": "blockTimestampLast", "type": "uint32"},
        ],
    },
    {
        "type": "function",
        "name": "token0",
        "stateMutability": "view",
        "inputs": [],
        "outputs": [{"name": "", "type": "address"}],
    },
]

def main():
    if not all([RPC_URL, TOKEN_A, TOKEN_B]):
        raise ValueError("Set RPC_URL, TOKEN_A and TOKEN_B.")

    w3 = Web3(Web3.HTTPProvider(RPC_URL))

    if not w3.is_connected():
        raise ConnectionError("RPC connection failed.")

    token_a = Web3.to_checksum_address(TOKEN_A)
    token_b = Web3.to_checksum_address(TOKEN_B)

    prices = []

    for factory_address in FACTORY_ADDRESSES:
        factory_address = factory_address.strip()

        if not factory_address:
            continue

        factory = w3.eth.contract(
            address=Web3.to_checksum_address(factory_address),
            abi=FACTORY_ABI,
        )

        try:
            pair_address = factory.functions.getPair(
                token_a, token_b
            ).call()

            if int(pair_address, 16) == 0:
                print(f"No pair in factory {factory_address}")
                continue

            pair = w3.eth.contract(
                address=Web3.to_checksum_address(pair_address),
                abi=PAIR_ABI,
            )

            reserve0, reserve1, _ = (
                pair.functions.getReserves().call()
            )

            actual_token0 = pair.functions.token0().call()

            if int(actual_token0, 16) == 0:
                print("Invalid token0 address; skipping pair.")
                continue

            if actual_token0.lower() == token_a.lower():
                reserve_a, reserve_b = reserve0, reserve1
            else:
                reserve_a, reserve_b = reserve1, reserve0

            if reserve_a == 0 or reserve_b == 0:
                print(f"Empty reserves: {pair_address}")
                continue

            # Raw-unit ratio; token decimals must be accounted
            # for before comparing human-readable token prices.
            raw_price = reserve_b / reserve_a

            prices.append({
                "factory": factory_address,
                "pair": pair_address,
                "raw_price": raw_price,
            })

            print(f"\nPair: {pair_address}")
            print(f"Raw tokenB/tokenA ratio: {raw_price:.10f}")

        except Exception as exc:
            print(f"Could not inspect {factory_address}: {exc}")

    if len(prices) < 2:
        print("\nNeed at least two valid pools to compare.")
        return

    lowest = min(prices, key=lambda item: item["raw_price"])
    highest = max(prices, key=lambda item: item["raw_price"])

    spread = (
        (highest["raw_price"] - lowest["raw_price"])
        / lowest["raw_price"]
    ) * 100

    print("\n--- Pool Price Comparison ---")
    print(f"Lowest ratio:  {lowest['raw_price']:.10f}")
    print(f"Highest ratio: {highest['raw_price']:.10f}")
    print(f"Raw spread:    {spread:.4f}%")
    print("This is not guaranteed arbitrage profit.")

if __name__ == "__main__":
    main()
