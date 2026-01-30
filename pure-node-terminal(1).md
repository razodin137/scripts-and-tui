const net = require('net');
const readline = require('readline');

// ANSI color codes
const colors = {
  reset: '\x1b[0m',
  bright: '\x1b[1m',
  green: '\x1b[32m',
  yellow: '\x1b[33m',
  blue: '\x1b[34m',
  red: '\x1b[31m',
  cyan: '\x1b[36m'
};

// In-memory database
const inventory = {
  'OST-001': { 
    breed: 'African Black', 
    age: 'Adult', 
    price: 15000,
    status: 'In Stock' 
  },
  'OST-002': { 
    breed: 'Blue Neck', 
    age: 'Juvenile', 
    price: 8000,
    status: 'Limited' 
  }
};

class TerminalSession {
  constructor(socket) {
    this.socket = socket;
    this.rl = readline.createInterface({
      input: socket,
      output: socket,
      prompt: `${colors.blue}ostrich-shop$ ${colors.reset}`
    });
  }

  write(text) {
    this.socket.write(text + '\n');
  }

  start() {
    this.write(`${colors.green}
=============================================
Welcome to Ostrich Terminal Shop v1.0.0
Type 'help' for available commands
=============================================\n${colors.reset}`);

    this.rl.prompt();

    this.rl.on('line', (line) => this.handleCommand(line));
    this.rl.on('close', () => this.socket.end());
  }

  handleCommand(line) {
    const cmd = line.trim().toLowerCase();
    const args = cmd.split(' ');

    try {
      switch(args[0]) {
        case 'help':
          this.showHelp();
          break;

        case 'list':
          this.listOstriches();
          break;

        case 'info':
          this.showInfo(args[1]);
          break;

        case 'price':
          this.showPricing();
          break;

        case 'buy':
          this.processPurchase(args[1]);
          break;

        case 'clear':
          this.socket.write('\x1B[2J\x1B[0f');
          break;

        case 'exit':
          this.write(`${colors.green}Thank you for visiting Ostrich Terminal Shop!${colors.reset}`);
          this.socket.end();
          return;

        default:
          this.write(`${colors.red}Command not found: ${args[0]}. Type 'help' for available commands.${colors.reset}`);
      }
    } catch (error) {
      this.write(`${colors.red}Error: ${error.message}${colors.reset}`);
    }

    this.rl.prompt();
  }

  showHelp() {
    this.write(`${colors.yellow}
Available commands:
  list        - Show available ostriches
  info <id>   - Get detailed information about an ostrich
  price       - Show current pricing
  buy <id>    - Purchase an ostrich
  clear       - Clear the screen
  exit        - Close connection${colors.reset}`);
  }

  listOstriches() {
    this.write(`${colors.green}\nAvailable Ostriches:${colors.reset}`);
    Object.entries(inventory).forEach(([id, ostrich]) => {
      this.write(`${colors.cyan}
  [${id}] ${ostrich.breed} (${ostrich.age})
    Status: ${ostrich.status}
    Price: $${ostrich.price.toLocaleString()}${colors.reset}`);
    });
  }

  showInfo(id) {
    if (!id) {
      this.write(`${colors.red}Usage: info <ostrich-id>${colors.reset}`);
      return;
    }

    const ostrich = inventory[id.toUpperCase()];
    if (!ostrich) {
      this.write(`${colors.red}Ostrich with ID ${id} not found${colors.reset}`);
      return;
    }

    this.write(`${colors.green}
Detailed Information for ${id}:
  Breed: ${ostrich.breed}
  Age: ${ostrich.age}
  Price: $${ostrich.price.toLocaleString()}
  Status: ${ostrich.status}
  
Care Requirements:
  - Daily water: 10L
  - Feed: 3.5kg grain mix
  - Space: 400 sq ft minimum
  - Temperature: 20-30°C${colors.reset}`);
  }

  showPricing() {
    this.write(`${colors.yellow}
Current Pricing:
  Adult Ostriches:    $15,000 - $25,000
  Juvenile Ostriches: $8,000 - $12,000
  Ostrich Eggs:       $300 - $500

* Prices may vary based on breed and availability
* Shipping not included${colors.reset}`);
  }

  processPurchase(id) {
    this.write(`${colors.yellow}
[PURCHASE SYSTEM]
Sorry, our payment system is currently under maintenance.
Please contact sales@ostrich.farm to complete your purchase.${colors.reset}`);
  }
}

const server = net.createServer((socket) => {
  console.log('Client connected!');
  
  const session = new TerminalSession(socket);
  session.start();

  socket.on('error', (err) => {
    console.error('Socket error:', err);
  });

  socket.on('end', () => {
    console.log('Client disconnected');
  });
});

const PORT = process.env.PORT || 23;
server.listen(PORT, '0.0.0.0', () => {
  console.log(`Terminal server listening on port ${PORT}`);
});
