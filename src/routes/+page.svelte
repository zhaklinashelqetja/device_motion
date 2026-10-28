<script>
    import { onMount } from 'svelte';
 
    let number = $state(6);
    let status = $state('Drücke Start und schüttle dein Handy');
    let started = $state(false);
    let rolling = $state(false);
 
    let gravity = { x: 0, y: 0, z: 0 };
    let lastShake = 0;
    let lastUpdate = 0;
 
    const SHAKE_THRESHOLD = 17;
    const SHAKE_COOLDOWN = 1200;
    const STABILITY_TIME = 500;
 
    let candidate = 6;
    let candidateSince = 0;
 
    // Ermittelt die Würfelseite anhand der Schwerkraft
    function getDiceSide(x, y, z) {
        const ax = Math.abs(x);
        const ay = Math.abs(y);
        const az = Math.abs(z);
 
        // Kleine Bewegungen ignorieren
        const values = [ax, ay, az];
        const sorted = [...values].sort((a, b) => b - a);
 
        if (sorted[0] - sorted[1] < 2.5) {
            return null;
        }
 
        if (az > ax && az > ay) {
            return z > 0 ? 6 : 1;
        }
 
        if (ax > ay && ax > az) {
            return x > 0 ? 2 : 5;
        }
 
        return y > 0 ? 3 : 4;
    }
 
    function handleOrientation(event) {
        // Fallback für Geräte ohne Gravity-Sensor
        if (event.beta === null || event.gamma === null) return;
 
        // Hier verwenden wir DeviceMotion für die Würfelseiten.
    }
 
    function handleMotion(event) {
        const acceleration = event.acceleration;
        const accelerationGravity = event.accelerationIncludingGravity;
 
        if (!accelerationGravity) return;
 
        const now = Date.now();
 
        // Shake erkennen
        if (acceleration) {
            const force = Math.sqrt(
                (acceleration.x ?? 0) ** 2 +
                (acceleration.y ?? 0) ** 2 +
                (acceleration.z ?? 0) ** 2
            );
 
            if (
                force > SHAKE_THRESHOLD &&
                now - lastShake > SHAKE_COOLDOWN
            ) {
                lastShake = now;
                rolling = true;
                status = '🎲 Würfel wird geworfen!';
                candidate = null;
                candidateSince = 0;
            }
        }
 
        // Schwerkraft glätten
        gravity.x = gravity.x * 0.8 +
            (accelerationGravity.x ?? 0) * 0.2;
 
        gravity.y = gravity.y * 0.8 +
            (accelerationGravity.y ?? 0) * 0.2;
 
        gravity.z = gravity.z * 0.8 +
            (accelerationGravity.z ?? 0) * 0.2;
 
        if (!rolling) return;
 
        // Nach dem Schütteln kurz warten
        if (now - lastShake < STABILITY_TIME) return;
 
        const side = getDiceSide(
            gravity.x,
            gravity.y,
            gravity.z
        );
 
        if (side === null) {
            candidate = null;
            return;
        }
 
        if (side !== candidate) {
            candidate = side;
            candidateSince = now;
            return;
        }
 
        // Erst übernehmen, wenn die Lage stabil bleibt
        if (now - candidateSince >= STABILITY_TIME) {
            number = side;
            rolling = false;
            status = '✅ Ergebnis: ' + number;
            lastUpdate = now;
        }
    }
 
    async function startSensors() {
        if (!window.isSecureContext) {
            status = 'HTTPS erforderlich';
            return;
        }
 
        if (!window.DeviceMotionEvent) {
            status = 'Device Motion wird nicht unterstützt';
            return;
        }
 
        try {
            // iPhone: Sensorberechtigung anfordern
            if (
                typeof DeviceMotionEvent.requestPermission === 'function'
            ) {
                const permission =
                    await DeviceMotionEvent.requestPermission();
 
                if (permission !== 'granted') {
                    status = 'Sensorzugriff verweigert';
                    return;
                }
            }
 
            if (!started) {
                window.addEventListener(
                    'devicemotion',
                    handleMotion
                );
 
                started = true;
            }
 
            status = '✅ Bereit! Schüttle dein Handy!';
 
        } catch (error) {
            status = 'Fehler beim Starten der Sensoren';
            console.error(error);
        }
    }
 
    onMount(() => {
        return () => {
            window.removeEventListener(
                'devicemotion',
                handleMotion
            );
        };
    });
</script>
 
<svelte:head>
    <title>Digital Dice 🎲</title>
</svelte:head>
 
<main>
    <h1>🎲 Digital Dice</h1>
 
    <p>Schüttle dein Handy und lege es auf eine Seite!</p>
 
    <div class="dice">
        {number}
    </div>
 
    <button onclick={startSensors} disabled={started}>
        {started ? 'Sensor aktiviert' : 'Start Device Motion'}
    </button>
 
    <p>{status}</p>
 
    <div class="info">
        <p>📱 Display oben → 6</p>
        <p>📱 Display unten → 1</p>
        <p>📱 Andere Seiten → 2, 3, 4, 5</p>
    </div>
</main>
 
<style>
    main {
        text-align: center;
        font-family: Arial, sans-serif;
        padding: 30px 15px;
    }
 
    h1 {
        font-size: 32px;
    }
 
    .dice {
        width: 160px;
        height: 160px;
        margin: 35px auto;
        display: flex;
        align-items: center;
        justify-content: center;
        font-size: 90px;
        font-weight: bold;
        background: #2563eb;
        color: white;
        border-radius: 25px;
        box-shadow: 0 8px 20px #0002;
    }
 
    button {
        padding: 14px 25px;
        background: #111827;
        color: white;
        border: none;
        border-radius: 10px;
        font-size: 16px;
        cursor: pointer;
    }
 
    button:disabled {
        background: #6b7280;
    }
 
    .info {
        margin-top: 30px;
        color: #555;
    }
</style>
 