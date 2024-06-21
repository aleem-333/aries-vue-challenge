<template>
  <div class="risk-reward-graph">
    <h1>Risk & Reward Graph</h1>

    <!-- User input section -->
    <div class="input-section">
      <div v-for="(option, index) in optionsData" :key="index" class="option-input">
        <label>Strike Price {{ index + 1 }}:</label>
        <input type="number" v-model="option.strike_price" @input="updateGraphData" />
        <label>Type:</label>
        <select v-model="option.type" @change="updateGraphData">
          <option :key="`type-${index}`" value="Call">Call</option>
          <option :key="`type-${index}-put`" value="Put">Put</option>
        </select>
        <label>Bid:</label>
        <input type="number" v-model="option.bid" @input="updateGraphData" />
        <label>Ask:</label>
        <input type="number" v-model="option.ask" @input="updateGraphData" />
        <label>Long/Short:</label>
        <select v-model="option.long_short" @change="updateGraphData">
          <option :key="`long_short-${index}`" value="long">Long</option>
          <option :key="`long_short-${index}-short`" value="short">Short</option>
        </select>
        <label>Expiration Date:</label>
        <input type="date" v-model="option.expiration_date" @input="updateGraphData" />
      </div>
    </div>

    <!-- Graph plotting area -->
    <div class="graph-container">
      <svg :width="graphWidth" :height="graphHeight">
        <!-- Axes and labels -->
        <line :x1="margin.left" :y1="graphHeight - margin.bottom" :x2="graphWidth - margin.right" :y2="graphHeight - margin.bottom" stroke="black" />
        <line :x1="margin.left" :y1="margin.top" :x2="margin.left" :y2="graphHeight - margin.bottom" stroke="black" />

        <!-- X-axis labels -->
        <g v-for="(label, index) in xLabels" :key="`xLabel-${index}`" class="x-axis-label">
          <text :x="scaleX(label.value)" :y="graphHeight - margin.bottom + 15" text-anchor="middle">{{ label.text }}</text>
        </g>

        <!-- Y-axis labels -->
        <g v-for="(label, index) in yLabels" :key="`yLabel-${index}`" class="y-axis-label">
          <text :x="margin.left - 10" :y="scaleY(label.value)" text-anchor="end">{{ label.text }}</text>
        </g>

        <!-- Plotting the line chart -->
        <polyline :points="linePoints" fill="none" stroke="blue" stroke-width="2" />

        <!-- Plotting profit/loss points with tooltips -->
        <g v-for="(point, index) in profitLossPoints" :key="`point-${index}`" class="data-point" @mouseover="showTooltip(point)" @mouseout="hideTooltip(point)">
          <circle :cx="scaleX(point.price)" :cy="scaleY(point.profitLoss)" r="4" fill="blue" />
          <text :x="scaleX(point.price)" :y="scaleY(point.profitLoss) - 10" text-anchor="middle">{{ point.label }}</text>
          <template v-if="point.showTooltip">
            <rect :x="scaleX(point.price) - 30" :y="scaleY(point.profitLoss) - 30" width="60" height="20" fill="white" stroke="black" />
            <text :x="scaleX(point.price)" :y="scaleY(point.profitLoss) - 15" text-anchor="middle">{{ point.tooltip }}</text>
          </template>
        </g>
      </svg>
    </div>

    <!-- Display max profit, max loss, and break even points -->
    <div class="summary">
      <p>Max Profit: {{ calculatedMaxProfit }}</p>
      <p>Max Loss: {{ calculatedMaxLoss }}</p>
      <p>Break Even Points: {{ calculatedBreakEvenPoints.join(', ') }}</p>
    </div>
  </div>
</template>

<script>
  export default {
    name: 'RiskRewardGraph',
    data() {
      return {
        // Define graph dimensions and margins
        graphWidth: 600,
        graphHeight: 400,
        margin: {
          top: 20,
          right: 20,
          bottom: 40,
          left: 50
        },
        // Initial options data structure for user input
        optionsData: [
          {
            strike_price: 100,
            type: 'Call',
            bid: 10.05,
            ask: 12.04,
            long_short: 'long',
            expiration_date: '2025-12-17'
          },
          {
            strike_price: 102.50,
            type: 'Call',
            bid: 12.10,
            ask: 14,
            long_short: 'long',
            expiration_date: '2025-12-17'
          },
          {
            strike_price: 103,
            type: 'Put',
            bid: 14,
            ask: 15.50,
            long_short: 'short',
            expiration_date: '2025-12-17'
          },
          {
            strike_price: 105,
            type: 'Put',
            bid: 16,
            ask: 18,
            long_short: 'long',
            expiration_date: '2025-12-17'
          }
        ]
      };
    },
    computed: {
      // Calculate profit/loss points for plotting
      profitLossPoints() {
        let points = [];

        // Determine the range of prices from the optionsData
        let minPrice = Math.min(...this.optionsData.map(option => option.strike_price));
        let maxPrice = Math.max(...this.optionsData.map(option => option.strike_price));

        // Iterate through each price point within the range
        for (let price = minPrice; price <= maxPrice; price += 0.5) {
          let profitLoss = this.calculateProfitLoss(price);
          points.push({ price, profitLoss, showTooltip: false }); // Added showTooltip property
        }

        return points;
      },
      // Calculate points for the line chart (connecting profit/loss points)
      linePoints() {
        return this.profitLossPoints.map(point => `${this.scaleX(point.price)},${this.scaleY(point.profitLoss)}`).join(' ');
      },
      // Calculate X-axis labels
      xLabels() {
        let labels = [];

        // Determine step size based on range
        let stepSize = Math.ceil((this.maxPrice - this.minPrice) / 10);

        for (let price = this.minPrice; price <= this.maxPrice; price += stepSize) {
          labels.push({
            value: price,
            text: price.toString()
          });
        }

        return labels;
      },
      // Calculate Y-axis labels
      yLabels() {
        let labels = [];
        let range = this.calculatedMaxProfit - this.calculatedMaxLoss;

        // Determine step size based on range
        let stepSize = range / 10;

        for (let i = 0; i <= 10; i++) {
          let value = this.calculatedMaxLoss + (i * stepSize);
          labels.push({
            value: value,
            text: value.toFixed(2)
          });
        }

        return labels;
      },
      // Calculate max profit from profitLossPoints
      calculatedMaxProfit() {
        return Math.max(...this.profitLossPoints.map(point => point.profitLoss), 0);
      },
      // Calculate max loss from profitLossPoints
      calculatedMaxLoss() {
        return Math.min(...this.profitLossPoints.map(point => point.profitLoss), 0);
      },
      // Calculate break even points (where profitLoss is close to 0)
      calculatedBreakEvenPoints() {
        return this.profitLossPoints.filter(point => Math.abs(point.profitLoss) < 0.01).map(point => point.price);
      },
      // Calculate min and max price from optionsData
      minPrice() {
        return Math.min(...this.optionsData.map(option => option.strike_price));
      },
      maxPrice() {
        return Math.max(...this.optionsData.map(option => option.strike_price));
      }
    },
    methods: {
      // Calculate profit/loss for a given underlying price
      calculateProfitLoss(price) {
        let totalProfitLoss = 0;

        this.optionsData.forEach(option => {
          if (option.type === 'Call') {
            if (price > option.strike_price) {
              totalProfitLoss += (price - option.strike_price - option.ask);
            } else {
              totalProfitLoss -= option.bid;
            }
          } else if (option.type === 'Put') {
            if (price < option.strike_price) {
              totalProfitLoss += (option.strike_price - price - option.ask);
            } else {
              totalProfitLoss -= option.bid;
            }
          }
        });

        return totalProfitLoss;
      },
      // Method to scale X-axis value based on graph width and margins
      scaleX(price) {
        let minPrice = Math.min(...this.optionsData.map(option => option.strike_price));
        let maxPrice = Math.max(...this.optionsData.map(option => option.strike_price));

        return this.margin.left + ((price - minPrice) / (maxPrice - minPrice) * (this.graphWidth - this.margin.left - this.margin.right));
      },
      // Method to scale Y-axis value based on graph height and margins
      scaleY(profitLoss) {
        let minProfitLoss = Math.min(0, this.calculatedMaxLoss);
        let maxProfitLoss = Math.max(0, this.calculatedMaxProfit);

        return this.graphHeight - this.margin.bottom - ((profitLoss - minProfitLoss) / (maxProfitLoss - minProfitLoss) * (this.graphHeight - this.margin.top - this.margin.bottom));
      },
      // Update graph data when user input changes
      updateGraphData() {
        // Recalculate graph related data when user input changes
        // For example, this.profitLossPoints = this.calculateProfitLoss();
        this.$nextTick(() => {
          // Update other computed properties as needed
        });
      },
      // Method to show tooltip on mouseover
      showTooltip(point) {
        point.showTooltip = true;
      },
      // Method to hide tooltip on mouseout
      hideTooltip(point) {
        point.showTooltip = false;
      }
    }
  };
</script>

<template>
  <div class="risk-reward-graph">
    <h1>Risk & Reward Graph</h1>

    <!-- User input section -->
    <div class="input-section">
      <div v-for="(option, index) in optionsData" :key="index" class="option-input">
        <label>Strike Price {{ index + 1 }}:</label>
        <input type="number" v-model="option.strike_price" @input="updateGraphData" />
        <label>Type:</label>
        <select v-model="option.type" @change="updateGraphData">
          <option :key="`type-${index}`" value="Call">Call</option>
          <option :key="`type-${index}-put`" value="Put">Put</option>
        </select>
        <label>Bid:</label>
        <input type="number" v-model="option.bid" @input="updateGraphData" />
        <label>Ask:</label>
        <input type="number" v-model="option.ask" @input="updateGraphData" />
        <label>Long/Short:</label>
        <select v-model="option.long_short" @change="updateGraphData">
          <option :key="`long_short-${index}`" value="long">Long</option>
          <option :key="`long_short-${index}-short`" value="short">Short</option>
        </select>
        <label>Expiration Date:</label>
        <input type="date" v-model="option.expiration_date" @input="updateGraphData" />
      </div>
    </div>

    <!-- Graph plotting area -->
    <div class="graph-container">
      <svg :width="graphWidth" :height="graphHeight">
        <!-- Axes and labels -->
        <line :x1="margin.left" :y1="graphHeight - margin.bottom" :x2="graphWidth - margin.right" :y2="graphHeight - margin.bottom" stroke="black" />
        <line :x1="margin.left" :y1="margin.top" :x2="margin.left" :y2="graphHeight - margin.bottom" stroke="black" />

        <!-- X-axis labels -->
        <g v-for="(label, index) in xLabels" :key="`xLabel-${index}`" class="x-axis-label">
          <text :x="scaleX(label.value)" :y="graphHeight - margin.bottom + 15" text-anchor="middle">{{ label.text }}</text>
        </g>

        <!-- Y-axis labels -->
        <g v-for="(label, index) in yLabels" :key="`yLabel-${index}`" class="y-axis-label">
          <text :x="margin.left - 10" :y="scaleY(label.value)" text-anchor="end">{{ label.text }}</text>
        </g>

        <!-- Plotting the line chart -->
        <polyline :points="linePoints" fill="none" stroke="blue" stroke-width="2" />

        <!-- Plotting profit/loss points with tooltips -->
        <g v-for="(point, index) in profitLossPoints" :key="`point-${index}`" class="data-point" @mouseover="showTooltip(point)" @mouseout="hideTooltip(point)">
          <circle :cx="scaleX(point.price)" :cy="scaleY(point.profitLoss)" r="4" fill="blue" />
          <text :x="scaleX(point.price)" :y="scaleY(point.profitLoss) - 10" text-anchor="middle">{{ point.label }}</text>
          <template v-if="point.showTooltip">
            <rect :x="scaleX(point.price) - 30" :y="scaleY(point.profitLoss) - 30" width="60" height="20" fill="white" stroke="black" />
            <text :x="scaleX(point.price)" :y="scaleY(point.profitLoss) - 15" text-anchor="middle">{{ point.tooltip }}</text>
          </template>
        </g>
      </svg>
    </div>

    <!-- Display max profit, max loss, and break even points -->
    <div class="summary">
      <p>Max Profit: {{ calculatedMaxProfit }}</p>
      <p>Max Loss: {{ calculatedMaxLoss }}</p>
      <p>Break Even Points: {{ calculatedBreakEvenPoints.join(', ') }}</p>
    </div>
  </div>
</template>

<script>
  export default {
    name: 'RiskRewardGraph',
    data() {
      return {
        // Define graph dimensions and margins
        graphWidth: 600,
        graphHeight: 400,
        margin: {
          top: 20,
          right: 20,
          bottom: 40,
          left: 50
        },
        // Initial options data structure for user input
        optionsData: [
          {
            strike_price: 100,
            type: 'Call',
            bid: 10.05,
            ask: 12.04,
            long_short: 'long',
            expiration_date: '2025-12-17'
          },
          {
            strike_price: 102.50,
            type: 'Call',
            bid: 12.10,
            ask: 14,
            long_short: 'long',
            expiration_date: '2025-12-17'
          },
          {
            strike_price: 103,
            type: 'Put',
            bid: 14,
            ask: 15.50,
            long_short: 'short',
            expiration_date: '2025-12-17'
          },
          {
            strike_price: 105,
            type: 'Put',
            bid: 16,
            ask: 18,
            long_short: 'long',
            expiration_date: '2025-12-17'
          }
        ]
      };
    },
    computed: {
      // Calculate profit/loss points for plotting
      profitLossPoints() {
        let points = [];

        // Determine the range of prices from the optionsData
        let minPrice = Math.min(...this.optionsData.map(option => option.strike_price));
        let maxPrice = Math.max(...this.optionsData.map(option => option.strike_price));

        // Iterate through each price point within the range
        for (let price = minPrice; price <= maxPrice; price += 0.5) {
          let profitLoss = this.calculateProfitLoss(price);
          points.push({ price, profitLoss, showTooltip: false }); // Added showTooltip property
        }

        return points;
      },
      // Calculate points for the line chart (connecting profit/loss points)
      linePoints() {
        return this.profitLossPoints.map(point => `${this.scaleX(point.price)},${this.scaleY(point.profitLoss)}`).join(' ');
      },
      // Calculate X-axis labels
      xLabels() {
        let labels = [];

        // Determine step size based on range
        let stepSize = Math.ceil((this.maxPrice - this.minPrice) / 10);

        for (let price = this.minPrice; price <= this.maxPrice; price += stepSize) {
          labels.push({
            value: price,
            text: price.toString()
          });
        }

        return labels;
      },
      // Calculate Y-axis labels
      yLabels() {
        let labels = [];
        let range = this.calculatedMaxProfit - this.calculatedMaxLoss;

        // Determine step size based on range
        let stepSize = range / 10;

        for (let i = 0; i <= 10; i++) {
          let value = this.calculatedMaxLoss + (i * stepSize);
          labels.push({
            value: value,
            text: value.toFixed(2)
          });
        }

        return labels;
      },
      // Calculate max profit from profitLossPoints
      calculatedMaxProfit() {
        return Math.max(...this.profitLossPoints.map(point => point.profitLoss), 0);
      },
      // Calculate max loss from profitLossPoints
      calculatedMaxLoss() {
        return Math.min(...this.profitLossPoints.map(point => point.profitLoss), 0);
      },
      // Calculate break even points (where profitLoss is close to 0)
      calculatedBreakEvenPoints() {
        return this.profitLossPoints.filter(point => Math.abs(point.profitLoss) < 0.01).map(point => point.price);
      },
      // Calculate min and max price from optionsData
      minPrice() {
        return Math.min(...this.optionsData.map(option => option.strike_price));
      },
      maxPrice() {
        return Math.max(...this.optionsData.map(option => option.strike_price));
      }
    },
    methods: {
      // Calculate profit/loss for a given underlying price
      calculateProfitLoss(price) {
        let totalProfitLoss = 0;

        this.optionsData.forEach(option => {
          if (option.type === 'Call') {
            if (price > option.strike_price) {
              totalProfitLoss += (price - option.strike_price - option.ask);
            } else {
              totalProfitLoss -= option.bid;
            }
          } else if (option.type === 'Put') {
            if (price < option.strike_price) {
              totalProfitLoss += (option.strike_price - price - option.ask);
            } else {
              totalProfitLoss -= option.bid;
            }
          }
        });

        return totalProfitLoss;
      },
      // Method to scale X-axis value based on graph width and margins
      scaleX(price) {
        let minPrice = Math.min(...this.optionsData.map(option => option.strike_price));
        let maxPrice = Math.max(...this.optionsData.map(option => option.strike_price));

        return this.margin.left + ((price - minPrice) / (maxPrice - minPrice) * (this.graphWidth - this.margin.left - this.margin.right));
      },
      // Method to scale Y-axis value based on graph height and margins
      scaleY(profitLoss) {
        let minProfitLoss = Math.min(0, this.calculatedMaxLoss);
        let maxProfitLoss = Math.max(0, this.calculatedMaxProfit);

        return this.graphHeight - this.margin.bottom - ((profitLoss - minProfitLoss) / (maxProfitLoss - minProfitLoss) * (this.graphHeight - this.margin.top - this.margin.bottom));
      },
      // Update graph data when user input changes
      updateGraphData() {
        // Recalculate graph related data when user input changes
        // For example, this.profitLossPoints = this.calculateProfitLoss();
        this.$nextTick(() => {
          // Update other computed properties as needed
        });
      },
      // Method to show tooltip on mouseover
      showTooltip(point) {
        point.showTooltip = true;
      },
      // Method to hide tooltip on mouseout
      hideTooltip(point) {
        point.showTooltip = false;
      }
    }
  };
</script>

<style scoped>
  /* Container styles */
  .risk-reward-graph {
    font-family: Arial, sans-serif;
    padding: 20px;
    background-color: #f0f0f0;
    border: 1px solid #ccc;
    border-radius: 5px;
  }

  /* Title styles */
  .risk-reward-graph h1 {
    font-size: 24px;
    margin-bottom: 20px;
    text-align: center;
  }

  /* Input section styles */
  .input-section {
    background-color: #fff;
    padding: 15px;
    border: 1px solid #ddd;
    border-radius: 5px;
    margin-bottom: 20px;
  }

  .option-input {
    display: flex;
    flex-wrap: wrap;
    margin-bottom: 10px;
  }

  .option-input label {
    margin-right: 10px;
    font-weight: bold;
  }

  .option-input input,
  .option-input select {
    margin-right: 10px;
    padding: 8px; /* Adjust padding for input elements */
    border: 1px solid #ccc;
    border-radius: 3px;
    width: 120px; /* Set a fixed width for consistency */
    box-sizing: border-box; /* Ensure padding and border are included in width */
    font-size: 14px; /* Adjust font size */
  }

  /* Graph container styles */
  .graph-container {
    border: 1px solid #ccc;
    border-radius: 5px;
    overflow: hidden; /* Ensures no overflow of graph */
    height: 400px;
  }

  .graph-container svg {
    width: 100%;
    height: 100%;
  }

  /* Axes and labels styles */
  .x-axis-label text, .y-axis-label text {
    font-size: 12px;
    fill: #666;
  }

  /* Data point styles */
  .data-point circle {
    fill: blue;
    stroke: white;
    stroke-width: 2;
  }

  .data-point text {
    font-size: 12px;
    text-anchor: middle;
    fill: blue;
  }

  /* Tooltip styles */
  .data-point rect {
    fill: white;
    stroke: black;
  }

  .data-point text {
    font-size: 12px;
    text-anchor: middle;
    fill: black;
  }

  /* Summary section styles */
  .summary {
    background-color: #fff;
    border: 1px solid #ddd;
    border-radius: 5px;
    padding: 10px;
  }

  .summary p {
    margin: 5px 0;
  }
</style>


