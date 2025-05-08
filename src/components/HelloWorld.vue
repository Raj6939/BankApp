<template>
    <div class="bank-container">
        <header class="bank-header">
            <div class="header-content">
                <div class="logo-section">
                    <div class="bank-logo">ABC</div>
                    <h1>Bank ABC</h1>
                </div>
                <p class="tagline">Your Trusted Banking Partner</p>
                <div class="security-badge">
                    <span class="security-icon">🔒</span>
                    Secure Banking Portal
                </div>
            </div>
            <div class="header-decoration"></div>
        </header>
        
        <div class="main-content">
            <div class="welcome-message">
                <h2>Welcome to Digital Banking</h2>
                <p>Experience secure and seamless banking at your fingertips</p>
            </div>

            <div class="qr-section">
                <div class="card">
                    <div class="card-header">
                        <div class="icon-circle">
                            <span class="qr-icon">📱</span>
                        </div>
                        <h2>Quick Account Access</h2>
                    </div>
                    <p class="subtitle">Scan the QR code with your banking app</p>
                    
                    <div class="qr-container">
                        <div class="qr-frame">
                            <canvas ref="qrCanvas" class="qr-code"></canvas>
                            <div class="qr-corners"></div>
                        </div>
                    </div>

                    <div class="connection-status" :class="wsStatusClass">
                        <span class="status-dot"></span>
                        {{ wsStatusMessage }}
                    </div>

                    <div class="security-info">
                        <div class="security-item">
                            <span class="security-icon">🔐</span>
                            End-to-End Encrypted
                        </div>
                        <div class="security-item">
                            <span class="security-icon">⚡</span>
                            Instant Access
                        </div>
                    </div>
                </div>
            </div>

            <!-- Authentication Success Card -->
            <div v-if="receivedData" class="data-container card fade-in">
                <div class="success-animation">
                    <div class="success-checkmark">✔️</div>
                </div>
                <h3>Authentication Successful</h3>
                <div class="data-details">
                    <div class="progress-bar">
                        <div class="progress-fill"></div>
                    </div>
                    <div class="details-content">
                        <p v-for="(value, key) in receivedData" :key="key">
                            <strong>{{ key }}:</strong> {{ value }}
                        </p>
                    </div>
                </div>
            </div>
        </div>

        <footer class="bank-footer">
            <div class="footer-content">
                <div class="footer-section">
                    <h4>Secure Banking</h4>
                    <p>Protected by Advanced Encryption</p>
                </div>
                <div class="footer-section">
                    <h4>24/7 Support</h4>
                    <p>Always Here to Help</p>
                </div>
                <div class="footer-section">
                    <h4>Global Access</h4>
                    <p>Bank from Anywhere</p>
                </div>
            </div>
            <div class="footer-bottom">
                <p>© 2024 Bank ABC. All rights reserved.</p>
            </div>
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
    position: relative;
    overflow: hidden;
}

.header-content {
    position: relative;
    z-index: 2;
    max-width: 1200px;
    margin: 0 auto;
    text-align: center;
}

.header-decoration {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: linear-gradient(135deg, rgba(255,255,255,0.1) 0%, rgba(255,255,255,0) 100%);
    transform: skewY(-6deg);
    transform-origin: top left;
}

.logo-section {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 1rem;
    margin-bottom: 1rem;
}

.bank-logo {
    background: rgba(255, 255, 255, 0.2);
    border-radius: 12px;
    padding: 0.8rem;
    font-size: 1.5rem;
    font-weight: bold;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.security-badge {
    display: inline-flex;
    align-items: center;
    background: rgba(255, 255, 255, 0.1);
    padding: 0.5rem 1rem;
    border-radius: 20px;
    margin-top: 1rem;
    font-size: 0.9rem;
    gap: 0.5rem;
}

.welcome-message {
    text-align: center;
    margin-bottom: 2rem;
}

.welcome-message h2 {
    color: #1a365d;
    font-size: 2rem;
    margin-bottom: 0.5rem;
}

.welcome-message p {
    color: #64748b;
    font-size: 1.1rem;
}

.card {
    background: white;
    border-radius: 16px;
    padding: 2rem;
    box-shadow: 0 10px 25px rgba(0, 0, 0, 0.1);
    margin-bottom: 2rem;
    text-align: center;
    transition: transform 0.3s ease;
}

.card:hover {
    transform: translateY(-5px);
}

.card-header {
    margin-bottom: 1.5rem;
}

.icon-circle {
    width: 60px;
    height: 60px;
    background: #f0f9ff;
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    margin: 0 auto 1rem;
}

.qr-icon {
    font-size: 2rem;
}

.qr-container {
    position: relative;
    padding: 2rem;
    margin: 2rem 0;
}

.qr-frame {
    position: relative;
    display: inline-block;
}

.qr-code {
    border-radius: 12px;
    padding: 1rem;
    background: white;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.05);
}

.qr-corners::before,
.qr-corners::after {
    content: '';
    position: absolute;
    width: 20px;
    height: 20px;
    border: 3px solid #1a365d;
}

.qr-corners::before {
    top: 0;
    left: 0;
    border-right: none;
    border-bottom: none;
}

.qr-corners::after {
    bottom: 0;
    right: 0;
    border-left: none;
    border-top: none;
}

.security-info {
    display: flex;
    justify-content: center;
    gap: 2rem;
    margin-top: 2rem;
    padding-top: 1rem;
    border-top: 1px solid #e2e8f0;
}

.security-item {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    color: #64748b;
    font-size: 0.9rem;
}

.security-icon {
    font-size: 1.2rem;
}

.connection-status {
    margin: 1rem auto;
    padding: 0.75rem 1.5rem;
    border-radius: 25px;
    font-size: 0.9rem;
    display: inline-flex;
    align-items: center;
    gap: 0.5rem;
    transition: all 0.3s ease;
}

.status-dot {
    width: 8px;
    height: 8px;
    border-radius: 50%;
    display: inline-block;
}

.status-connected {
    background-color: #ecfdf5;
}

.status-connected .status-dot {
    background-color: #059669;
    box-shadow: 0 0 0 3px rgba(5, 150, 105, 0.2);
}

.status-disconnected {
    background-color: #fef2f2;
}

.status-disconnected .status-dot {
    background-color: #dc2626;
    box-shadow: 0 0 0 3px rgba(220, 38, 38, 0.2);
}

.status-connecting {
    background-color: #fffbeb;
}

.status-connecting .status-dot {
    background-color: #d97706;
    box-shadow: 0 0 0 3px rgba(217, 119, 6, 0.2);
    animation: pulse 1.5s infinite;
}

.success-animation {
    margin-bottom: 1.5rem;
}

.success-checkmark {
    font-size: 3.5rem;
    color: #059669;
    animation: bounceIn 0.6s cubic-bezier(0.68, -0.55, 0.265, 1.55);
}

.data-details {
    background-color: #f8fafc;
    border-radius: 12px;
    overflow: hidden;
}

.progress-bar {
    height: 4px;
    background-color: #e2e8f0;
    margin-bottom: 1rem;
}

.progress-fill {
    height: 100%;
    background-color: #059669;
    animation: progressFill 1s ease-out forwards;
}

.details-content {
    padding: 1.5rem;
}

.details-content p {
    margin: 0.75rem 0;
    color: #4b5563;
    display: flex;
    justify-content: space-between;
    align-items: center;
}

.details-content strong {
    color: #1a365d;
}

.footer-content {
    max-width: 1200px;
    margin: 0 auto;
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 2rem;
    padding: 2rem;
}

.footer-section {
    text-align: center;
}

.footer-section h4 {
    color: white;
    margin-bottom: 0.5rem;
}

.footer-bottom {
    text-align: center;
    padding-top: 1.5rem;
    border-top: 1px solid rgba(255, 255, 255, 0.1);
}

.fade-in {
    animation: fadeIn 0.5s ease-out;
}

@keyframes pulse {
    0% { transform: scale(1); opacity: 1; }
    50% { transform: scale(1.2); opacity: 0.5; }
    100% { transform: scale(1); opacity: 1; }
}

@keyframes bounceIn {
    0% { transform: scale(0); }
    50% { transform: scale(1.2); }
    100% { transform: scale(1); }
}

@keyframes progressFill {
    from { width: 0; }
    to { width: 100%; }
}

@keyframes fadeIn {
    from { opacity: 0; transform: translateY(20px); }
    to { opacity: 1; transform: translateY(0); }
}

@media (max-width: 768px) {
    .footer-content {
        grid-template-columns: 1fr;
        gap: 1.5rem;
    }

    .security-info {
        flex-direction: column;
        gap: 1rem;
    }

    .welcome-message h2 {
        font-size: 1.5rem;
    }
}
</style>
