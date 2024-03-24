# Decentralized-Ride-Hailing-for-Autonomous-Vehicles
Build a WEB3-based ride-hailing platform for autonomous vehicles, ensuring secure transactions and data privacy.
from flask import Flask, request, jsonify
from web3 import Web3
import json

app = Flask(__name__)

# Connect to Ethereum blockchain (This is a testnet example)
w3 = Web3(Web3.HTTPProvider('https://rinkeby.infura.io/v3/YOUR_INFURA_PROJECT_ID'))
w3.eth.defaultAccount = w3.eth.account.privateKeyToAccount('YOUR_PRIVATE_KEY').address

# Load the compiled smart contract
with open('RidePaymentContract.json', 'r') as f:
    contract_abi = json.load(f)['abi']
contract_address = Web3.toChecksumAddress('YOUR_CONTRACT_ADDRESS')
ride_payment_contract = w3.eth.contract(address=contract_address, abi=contract_abi)

# Dummy database of vehicles and rides
vehicles = [{'id': 1, 'location': 'Downtown', 'status': 'available'}]
rides = []

@app.route('/request_ride', methods=['POST'])
def request_ride():
    data = request.json
    ride = {
        'id': len(rides) + 1,
        'user_id': data['user_id'],
        'pickup_location': data['pickup_location'],
        'destination': data['destination'],
        'status': 'pending',
    }
    rides.append(ride)
    match_ride(ride['id'])
    return jsonify(ride), 200

def match_ride(ride_id):
    ride = next((r for r in rides if r['id'] == ride_id), None)
    if ride:
        vehicle = next((v for v in vehicles if v['status'] == 'available'), None)
        if vehicle:
            ride['vehicle_id'] = vehicle['id']
            ride['status'] = 'matched'
            vehicle['status'] = 'on_ride'
            # Here you would call the smart contract to handle payment/security deposit
            # For simplicity, this step is omitted

@app.route('/complete_ride', methods=['POST'])
def complete_ride():
    data = request.json
    ride = next((r for r in rides if r['id'] == data['ride_id']), None)
    if ride:
        ride['status'] = 'completed'
        vehicle = next((v for v in vehicles if v['id'] == ride['vehicle_id']), None)
        if vehicle:
            vehicle['status'] = 'available'
            # Handle payment through smart contract
            tx_hash = ride_payment_contract.functions.completePayment(ride['id']).transact()
            tx_receipt = w3.eth.waitForTransactionReceipt(tx_hash)
            return jsonify({'status': 'Ride completed', 'transaction_receipt': tx_receipt}), 200
    return jsonify({'error': 'Ride not found'}), 404

if __name__ == '__main__':
    app.run(debug=True, port=5000)
