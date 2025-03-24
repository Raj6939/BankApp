<template>
    <div class="container">
        <h2>Scan the QR Code</h2>
        <canvas ref="qrCanvas" class="qr-code"></canvas>

        <!-- Tick Mark & Data Display -->
         <br>
        <div v-if="receivedData" class="data-container">
            <div class="success-checkmark">✔️</div>
            <h3>Data Received:</h3>
            <p v-for="(value, key) in receivedData" :key="key">
                <strong>{{ key }}:</strong> {{ value }}
            </p>
        </div>
    </div>
</template>

<script>
import QRCode from "qrcode";
import config from "../../config";
export default {
    data() {
        return {
            ws: null,
            clientId: Math.random().toString(36).substr(2, 9), // Unique Client ID
            receivedData: null,
            wsStatus: "connecting", // WebSocket status: "connecting" | "connected" | "disconnected"
        };
    },
    computed: {
        wsStatusMessage() {
            return this.wsStatus === "connected"
                ? "🟢 Connected"
                : this.wsStatus === "disconnected"
                ? "🔴 Disconnected (Reconnecting...)"
                : "🟡 Connecting...";
        },
        wsStatusClass() {
            return {
                "status-connected": this.wsStatus === "connected",
                "status-disconnected": this.wsStatus === "disconnected",
                "status-connecting": this.wsStatus === "connecting",
            };
        }
    },
    methods: {
        async generateQR() {
            try {
                const qrData = JSON.stringify({
                    wsUrl: config.websocketUrl,
                    clientId: this.clientId
                });
                await QRCode.toCanvas(this.$refs.qrCanvas, qrData);
            } catch (error) {
                console.error("QR Code generation failed:", error);
            }
        },
        setupWebSocket() {
            if (this.ws) {
                this.ws.close();
            }

            this.ws = new WebSocket(config.websocketUrl);

            this.ws.onopen = () => {
                console.log("WebSocket connected.");
                this.wsStatus = "connected";
                this.sendMessage({ type: "register", clientId: this.clientId });
            };

            this.ws.onmessage = (event) => {
                console.log("Raw WebSocket Message:", event.data); // Debugging: Log raw data
                try {
                    let data = JSON.parse(event.data);
                    console.log("Parsed WebSocket Data:", data); // Debugging: Log parsed data
                    if (data.type === "receivedData") {
                        this.receivedData = data.payload;
                        console.log("Successfully received data:", this.receivedData); // Confirm receipt
                    } else {
                        console.warn("Unexpected WebSocket message type:", data.type);
                    }
                } catch (error) {
                    console.error("Error parsing WebSocket message:", error);
                }
            };


            this.ws.onerror = (error) => {
                console.error("WebSocket Error:", error);
            };

            this.ws.onclose = () => {
                console.warn("WebSocket closed. Attempting to reconnect...");
                this.wsStatus = "disconnected";
                setTimeout(() => {
                    this.wsStatus = "connecting";
                    this.setupWebSocket();
                }, 3000);
            };
        },
        sendMessage(message) {
            if (this.ws && this.ws.readyState === WebSocket.OPEN) {
                this.ws.send(JSON.stringify(message));
            } else {
                console.warn("WebSocket is not open. Message not sent.");
            }
        }
    },
    mounted() {
        this.generateQR();
        this.setupWebSocket();
    },
    beforeDestroy() {
        if (this.ws) {
            this.ws.close();
        }
    }
};
</script>

<style scoped>
.container {
    text-align: center;
    font-family: Arial, sans-serif;
}
.qr-code {
    margin: 20px auto;
}
.data-container {
    margin-top: 20px;
    padding: 10px;
    border: 2px solid green;
    background-color: #eaffea;
    display: inline-block;
    border-radius: 8px;
}
.success-checkmark {
    font-size: 50px;
    color: green;
}
.received-data {
    white-space: pre-wrap;
    word-wrap: break-word;
    text-align: left;
    background-color: #f8f8f8;
    padding: 10px;
    border-radius: 5px;
    border: 1px solid #ddd;
}
.status-connected {
    color: green;
    font-weight: bold;
}
.status-disconnected {
    color: red;
    font-weight: bold;
}
.status-connecting {
    color: orange;
    font-weight: bold;
}
</style>
