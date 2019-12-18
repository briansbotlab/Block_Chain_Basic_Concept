# Block_Chain_Basic_Concept

<p>區塊鏈(Block Chain)，是一種由資料加解密技術、工作量證明演算法及共識演算法這幾樣技術的結合，進而達到去中心化的資料儲存。<p>


## 程式碼說明
<p>第一部分:引入相關套件。</p>
<pre>
<code>
  import hashlib 
  import json 
  from textwrap import dedent 
  from time import time 
  from uuid import uuid4 
  from flask import Flask, jsonify, request 
  from urllib.parse import urlparse
  import requests
</code>
</pre>

<p>第二部分:建立一個類別(Class)裡面有提供區塊鏈的核心概念的功能。</p>
<pre>
<code>
  class Blockchain(object): 
    def __init__(self): 
        self.current_transactions = [] 
        self.chain = [] 
        self.nodes = set()
        # Create the genesis block 
        self.new_block(previous_hash=1, proof=100)
    def register_node(self, address): 
    #Add a new node to the list of nodes 
 
        parsed_url = urlparse(address) 
        self.nodes.add(parsed_url.netloc) 
        
    def new_block(self, proof, previous_hash=None):

    #生成新塊

        block = { 'index': len(self.chain) + 1,
        'timestamp': time(),
        'transactions': self.current_transactions,
        'proof': proof,
        'previous_hash': previous_hash or self.hash(self.chain[-1]),
        }
        # Reset the current list of transactions 
        self.current_transactions = [] 
        self.chain.append(block) 
        return block 
    def new_transaction(self, sender, recipient, amount): 
    #生成新交易資訊，資訊將加入到下一個待挖的區塊中

        self.current_transactions.append({ 'sender': sender,
        'recipient': recipient,
        'amount': amount, }) 
        return self.last_block['index'] + 1 
    @property 
    def last_block(self): 
        return self.chain[-1] 
    @staticmethod
    def hash(block):
        block_string = json.dumps(block, sort_keys=True).encode() 
        return hashlib.sha256(block_string).hexdigest()

    def proof_of_work(self, last_proof):
    #簡單的工作量證明
    # - 查詢一個 p' 使得 hash(pp') 以4個0開頭 - p 是上一個塊的證明, p' 是當前的證明 

        proof = 0 
        while self.valid_proof(last_proof, proof) is False: 
            proof += 1 
            return proof 
    @staticmethod 
    def valid_proof(last_proof, proof):
    #驗證證明 是否hash(last_proof, proof)以4個0開頭?
    
        guess = f'{last_proof}{proof}'.encode() 
        guess_hash = hashlib.sha256(guess).hexdigest()
        return guess_hash[:4] == "0000" 
    def valid_chain(self, chain): 
    #Determine if a given blockchain is valid 
        last_block = chain[0] 
        current_index = 1 
        while current_index < len(chain): 
            block = chain[current_index] 
            print(f'{last_block}') 
            print(f'{block}') 
            print("\n-----------\n") 
            # Check that the hash of the block is correct 
            if block['previous_hash'] != self.hash(last_block): 
                return False 
            # Check that the Proof of Work is correct 
            if not self.valid_proof(last_block['proof'], block['proof']): 
                return False 
            last_block = block 
            current_index += 1 
            
        return True
    def resolve_conflicts(self): 
    #共識演算法解決衝突 使用網路中最長的鏈.
        neighbours = self.nodes 
        new_chain = None 
        # We're only looking for chains longer than ours 
        max_length = len(self.chain) 
    # Grab and verify the chains from all the nodes in our network 
        for node in neighbours: 
            response = requests.get(f'http://{node}/chain') 
            if response.status_code == 200: 
                length = response.json()['length'] 
                chain = response.json()['chain'] 
                # Check if the length is longer and the chain is valid 
                if length > max_length and self.valid_chain(chain): 
                    max_length = length 
                    new_chain = chain 
        # Replace our chain if we discovered a new, valid chain longer than ours 
        if new_chain: 
            self.chain = new_chain 
            return True 
        return False 
</code>
</pre>

<p>第三部分:使用Flask框架，將方法變成可用瀏覽器網址列呼叫的形式。</p>
<pre>
<code>
  @app.route('/chain', methods=['GET']) 
def full_chain(): 
    response = {
    'chain': blockchain.chain,
    'length': len(blockchain.chain),
    } 
    return jsonify(response), 200 
    
@app.route('/transactions/new', methods=['POST']) 
def new_transaction(): 
    values = request.get_json() 
    
    #Check that the required fields are in the POST'ed data 
    required = ['sender', 'recipient', 'amount'] 
    if not all(k in values for k in required): 
        return 'Missing values', 400 
    #Create a new Transaction 
    index = blockchain.new_transaction(values['sender'],
    values['recipient'],
    values['amount']) 
    response = {'message': f'Transaction will be added to Block{index}'} 
    return jsonify(response), 201 
    
@app.route('/mine', methods=['GET']) 
def mine(): 
#We run the proof of work algorithm to get the next proof... 
    last_block = blockchain.last_block 
    last_proof = last_block['proof'] 
    proof = blockchain.proof_of_work(last_proof) 
    # 給工作量證明的節點提供獎勵. 
    # 傳送者為 "0" 表明是新挖出的幣 
    blockchain.new_transaction( sender="0", recipient=node_identifier, amount=1, ) 
    # Forge the new Block by adding it to the chain 
    block = blockchain.new_block(proof) 
    response = { 'message': "New Block Forged",
    'index': block['index'],
    'transactions': block['transactions'],
    'proof': block['proof'],
    'previous_hash': block['previous_hash'], } 
    return jsonify(response), 200 
    
@app.route('/nodes/register', methods=['POST']) 
def register_nodes(): 
    values = request.get_json() 
    nodes = values.get('nodes') 
    if nodes is None: 
        return "Error: Please supply a valid list of nodes", 400 
        
    for node in nodes: 
        blockchain.register_node(node) 
    response = { 'message': 'New nodes have been added',
    'total_nodes': list(blockchain.nodes),
    } 
    return jsonify(response), 201 
    
@app.route('/nodes/resolve', methods=['GET']) 
def consensus(): 
    replaced = blockchain.resolve_conflicts() 
    if replaced: 
        response = { 'message': 'Our chain was replaced',
        'new_chain': blockchain.chain 
        } 
    else: 
        response = { 'message': 'Our chain is authoritative',
        'chain': blockchain.chain 
        } 
    return jsonify(response), 200 
</code>
</pre>

<p>第四部分:使用Flask框架建立基本區塊鏈。</p>
<pre>
<code>
  #Instantiate our Node
  app = Flask(__name__) 
  #Generate a globally unique address for this node 
  node_identifier = str(uuid4()).replace('-', '') 
  #Instantiate the Blockchain 
  blockchain = Blockchain() 
</code>
</pre>

<p>第五部分:定義程式起始區段，運行Flask並設置port為5000。</p>
<pre>
<code>
 if __name__ == '__main__': 
    app.run(host='127.0.0.1', port=5000) 
</code>
</pre>




