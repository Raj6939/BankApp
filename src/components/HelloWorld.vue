<template>
    <div class="bank-container">
        <header class="bank-header">
            <h1>Bank ABC</h1>
            <p class="tagline">Your Trusted Banking Partner</p>
        </header>
        
        <div class="main-content">
            <div class="qr-section">
                <div class="card">
                    <h2>Scan to Access Your Account</h2>
                    <p class="subtitle">Use your banking app to scan the QR code</p>
                    <canvas ref="qrCanvas" class="qr-code"></canvas>
                    <div class="connection-status" :class="wsStatusClass">
                        {{ wsStatusMessage }}
                    </div>
                </div>
            </div>

            <!-- Tick Mark & Data Display -->
            <div v-if="receivedData" class="data-container card">
                <div class="success-checkmark">✔️</div>
                <h3>Authentication Successful</h3>
                <div class="data-details">
                    <p v-for="(value, key) in receivedData" :key="key">
                        <strong>{{ key }}:</strong> {{ value }}
                    </p>
                </div>
            </div>
        </div>

        <footer class="bank-footer">
            <p>© 2024 Bank ABC. All rights reserved.</p>
            <p>Secure Banking • 24/7 Support • Protected by Advanced Encryption</p>
        </footer>
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
.bank-container {
    min-height: 100vh;
    display: flex;
    flex-direction: column;
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
    background-color: #f5f7fa;
    color: #2c3e50;
}

.bank-header {
    background-color: #1a365d;
    color: white;
    padding: 2rem;
    text-align: center;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.bank-header h1 {
    margin: 0;
    font-size: 2.5rem;
    font-weight: 600;
}

.tagline {
    margin: 0.5rem 0 0;
    font-size: 1.1rem;
    opacity: 0.9;
}

.main-content {
    flex: 1;
    padding: 2rem;
    max-width: 1200px;
    margin: 0 auto;
    width: 100%;
    box-sizing: border-box;
}

.card {
    background: white;
    border-radius: 12px;
    padding: 2rem;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    margin-bottom: 2rem;
    text-align: center;
}

.qr-section {
    max-width: 500px;
    margin: 0 auto;
}

.qr-code {
    margin: 1.5rem auto;
    padding: 1rem;
    background: white;
    border-radius: 8px;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
}

.subtitle {
    color: #666;
    margin: 0.5rem 0 1.5rem;
}

.connection-status {
    margin-top: 1rem;
    padding: 0.5rem;
    border-radius: 4px;
    font-size: 0.9rem;
}

.status-connected {
    color: #059669;
    background-color: #ecfdf5;
}

.status-disconnected {
    color: #dc2626;
    background-color: #fef2f2;
}

.status-connecting {
    color: #d97706;
    background-color: #fffbeb;
}

.data-container {
    max-width: 500px;
    margin: 2rem auto;
    text-align: left;
}

.success-checkmark {
    font-size: 3rem;
    color: #059669;
    margin-bottom: 1rem;
}

.data-details {
    background-color: #f8fafc;
    padding: 1rem;
    border-radius: 8px;
    margin-top: 1rem;
}

.data-details p {
    margin: 0.5rem 0;
    color: #4b5563;
}

.data-details strong {
    color: #1a365d;
}

.bank-footer {
    background-color: #1a365d;
    color: white;
    text-align: center;
    padding: 1.5rem;
    margin-top: auto;
}

.bank-footer p {
    margin: 0.5rem 0;
    font-size: 0.9rem;
    opacity: 0.9;
}
</style>
