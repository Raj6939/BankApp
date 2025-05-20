<template>
    <div class="container">
        <!-- QR Code View -->
        <div v-if="!receivedData" class="qr-view" :class="{ 'fade-out': receivedData }">
            <div class="brand">
                <div class="logo-pulse"></div>
                <span class="brand-text">SecureConnect</span>
            </div>
            
            <div class="context-message">
                <div class="context-icon">🏦</div>
                <h2>Secure Data Request</h2>
                <p>Scan with your banking app to securely share selected information</p>
            </div>

            <div class="qr-wrapper">
                <div class="secure-border">
                    <canvas ref="qrCanvas" class="qr-code"></canvas>
                </div>
            </div>
            
            <div class="status-indicator" :class="wsStatusClass">
                {{ wsStatusMessage }}
            </div>
            
            <div class="security-features">
                <div class="security-item">
                    <span class="security-icon">🔒</span>
                    End-to-end encrypted
                </div>
                <div class="security-item">
                    <span class="security-icon">👤</span>
                    PIN protected
                </div>
                <div class="security-item">
                    <span class="security-icon">✨</span>
                    Selective sharing
                </div>
            </div>

            <div class="info-tooltip">
                Your data remains private. Share only what you choose.
            </div>
        </div>

        <!-- Enhanced Data View -->
        <div v-if="receivedData" class="data-view">
            <div class="success-indicator">
                <div class="success-circle">
                    <svg class="checkmark" viewBox="0 0 24 24">
                        <path class="checkmark-path" d="M3.5 12.5L9 18L20 7" fill="none" stroke="currentColor" stroke-width="2"/>
                    </svg>
                </div>
                <span class="success-text">Verified</span>
                <p class="success-subtitle">Data shared securely</p>
            </div>

            <div class="data-grid">
                <div v-for="(value, key) in receivedData" :key="key" class="data-item">
                    <div class="data-value">{{ value }}</div>
                    <div class="data-label">{{ formatLabel(key) }}</div>
                </div>
            </div>

            <div class="verification-details">
                <div class="timestamp">
                    <span class="detail-icon">🕒</span>
                    {{ currentDateTime }}
                </div>
                <div class="session-id">
                    <span class="detail-icon">🔐</span>
                    Session #{{ clientId.slice(0, 8) }}
                </div>
            </div>
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
            currentDateTime: ""
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
        },
        formatLabel(key) {
            return key.split('_')
                .map(word => word.charAt(0).toUpperCase() + word.slice(1))
                .join(' ');
        },
        updateDateTime() {
            const now = new Date();
            const options = { 
                year: 'numeric', 
                month: 'long', 
                day: 'numeric',
                hour: '2-digit',
                minute: '2-digit',
                second: '2-digit'
            };
            this.currentDateTime = now.toLocaleDateString('en-US', options);
        }
    },
    mounted() {
        this.generateQR();
        this.setupWebSocket();
        this.updateDateTime();
        setInterval(this.updateDateTime, 1000);
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
    min-height: 100vh;
    display: flex;
    align-items: center;
    justify-content: center;
    background: linear-gradient(135deg, #f8fafc 0%, #ffffff 100%);
    padding: 20px;
}

/* Brand Styling */
.brand {
    display: flex;
    align-items: center;
    gap: 12px;
    margin-bottom: 32px;
}

.logo-pulse {
    width: 40px;
    height: 40px;
    background: linear-gradient(135deg, #3b82f6 0%, #2563eb 100%);
    border-radius: 12px;
    position: relative;
}

.logo-pulse::after {
    content: '';
    position: absolute;
    top: -4px;
    left: -4px;
    right: -4px;
    bottom: -4px;
    border-radius: 14px;
    background: linear-gradient(135deg, #3b82f6 0%, #2563eb 100%);
    opacity: 0.3;
    animation: pulse 2s infinite;
}

.brand-text {
    font-size: 24px;
    font-weight: 600;
    background: linear-gradient(135deg, #1e293b 0%, #334155 100%);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
}

/* QR Code Styling */
.qr-view {
    text-align: center;
    max-width: 400px;
    width: 100%;
}

.qr-wrapper {
    position: relative;
    padding: 24px;
    background: white;
    border-radius: 24px;
    box-shadow: 
        0 4px 6px rgba(0, 0, 0, 0.05),
        0 10px 15px -3px rgba(0, 0, 0, 0.1);
    margin-bottom: 24px;
    transition: transform 0.3s ease;
}

.qr-wrapper:hover {
    transform: translateY(-2px);
}

.secure-border {
    position: relative;
    padding: 1px;
    background: linear-gradient(135deg, #3b82f6 0%, #2563eb 100%);
    border-radius: 16px;
    display: inline-block;
}

.qr-code {
    display: block;
    border-radius: 15px;
    background: white;
}

/* Status Indicator */
.status-indicator {
    display: inline-flex;
    align-items: center;
    padding: 10px 20px;
    border-radius: 40px;
    font-size: 14px;
    font-weight: 500;
    transition: all 0.3s ease;
}

.status-connected {
    background: #dcfce7;
    color: #166534;
}

.status-disconnected {
    background: #fee2e2;
    color: #991b1b;
}

.status-connecting {
    background: #fef3c7;
    color: #92400e;
    animation: pulse 2s infinite;
}

/* Security Hint */
.security-hint {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 8px;
    margin-top: 16px;
    color: #64748b;
    font-size: 14px;
}

.security-icon {
    font-size: 16px;
}

/* Enhanced Data View */
.data-view {
    width: 100%;
    max-width: 600px;
    animation: slideUp 0.5s ease-out;
}

.success-indicator {
    text-align: center;
    margin-bottom: 32px;
}

.success-circle {
    width: 64px;
    height: 64px;
    background: #22c55e;
    border-radius: 50%;
    margin: 0 auto 16px;
    display: flex;
    align-items: center;
    justify-content: center;
    animation: scaleIn 0.5s ease-out;
}

.checkmark {
    width: 32px;
    height: 32px;
    color: white;
}

.checkmark-path {
    stroke-dasharray: 100;
    stroke-dashoffset: 100;
    animation: drawCheck 0.6s ease-out forwards 0.3s;
}

.success-text {
    font-size: 20px;
    font-weight: 600;
    color: #166534;
    opacity: 0;
    animation: fadeIn 0.3s ease-out forwards 0.6s;
}

.data-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
    gap: 16px;
    margin-bottom: 24px;
}

.data-item {
    background: white;
    padding: 24px;
    border-radius: 16px;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.05);
    transition: all 0.3s ease;
    opacity: 0;
    animation: fadeIn 0.3s ease-out forwards;
    animation-delay: calc(var(--index, 0) * 0.1s + 0.8s);
}

.data-item:hover {
    transform: translateY(-2px);
    box-shadow: 0 6px 8px rgba(0, 0, 0, 0.08);
}

.data-value {
    font-size: 24px;
    font-weight: 600;
    color: #1e293b;
    margin-bottom: 8px;
}

.data-label {
    font-size: 14px;
    color: #64748b;
}

.verification-details {
    display: flex;
    flex-direction: column;
    gap: 12px;
    text-align: center;
    color: #64748b;
    font-size: 14px;
    opacity: 0;
    animation: fadeIn 0.3s ease-out forwards 1.2s;
}

.detail-icon {
    margin-right: 8px;
}

/* Animations */
@keyframes pulse {
    0% {
        opacity: 0.3;
        transform: scale(1);
    }
    50% {
        opacity: 0.1;
        transform: scale(1.1);
    }
    100% {
        opacity: 0.3;
        transform: scale(1);
    }
}

@keyframes scaleIn {
    from {
        transform: scale(0);
    }
    to {
        transform: scale(1);
    }
}

@keyframes drawCheck {
    to {
        stroke-dashoffset: 0;
    }
}

@keyframes slideUp {
    from {
        opacity: 0;
        transform: translateY(20px);
    }
    to {
        opacity: 1;
        transform: translateY(0);
    }
}

@keyframes fadeIn {
    from {
        opacity: 0;
        transform: translateY(10px);
    }
    to {
        opacity: 1;
        transform: translateY(0);
    }
}

.fade-out {
    animation: fadeOut 0.3s ease-out forwards;
}

@keyframes fadeOut {
    from {
        opacity: 1;
        transform: translateY(0);
    }
    to {
        opacity: 0;
        transform: translateY(-20px);
    }
}

@media (max-width: 480px) {
    .data-grid {
        grid-template-columns: 1fr;
    }
    
    .verification-details {
        flex-direction: column;
        align-items: center;
    }

    .security-features {
        padding: 0 16px;
    }

    .info-tooltip {
        margin: 24px 16px 0;
    }
}

.context-message {
    margin-bottom: 24px;
    animation: fadeIn 0.5s ease-out;
}

.context-icon {
    font-size: 32px;
    margin-bottom: 16px;
}

.context-message h2 {
    font-size: 24px;
    font-weight: 600;
    color: #1e293b;
    margin-bottom: 8px;
}

.context-message p {
    color: #64748b;
    font-size: 16px;
    line-height: 1.5;
}

.security-features {
    display: flex;
    flex-direction: column;
    gap: 12px;
    margin-top: 24px;
}

.security-item {
    display: flex;
    align-items: center;
    gap: 8px;
    color: #64748b;
    font-size: 14px;
    padding: 8px 16px;
    background: rgba(255, 255, 255, 0.8);
    border-radius: 20px;
    backdrop-filter: blur(8px);
    transition: transform 0.2s ease;
}

.security-item:hover {
    transform: translateX(4px);
}

.info-tooltip {
    margin-top: 24px;
    padding: 12px 20px;
    background: #f8fafc;
    border-left: 4px solid #3b82f6;
    border-radius: 8px;
    color: #64748b;
    font-size: 14px;
    text-align: left;
    animation: slideIn 0.5s ease-out;
}

.success-subtitle {
    color: #64748b;
    font-size: 14px;
    margin-top: 4px;
    opacity: 0;
    animation: fadeIn 0.3s ease-out forwards 0.8s;
}

@keyframes slideIn {
    from {
        opacity: 0;
        transform: translateX(-20px);
    }
    to {
        opacity: 1;
        transform: translateX(0);
    }
}
</style>
