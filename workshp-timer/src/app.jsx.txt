import { useState, useEffect, useRef } from 'react';

// Firebase config
const firebaseConfig = {
  databaseURL: "https://time-koper-default-rtdb.europe-west1.firebasedatabase.app",
};

const dbUrl = firebaseConfig.databaseURL;

async function setData(path, data) {
  await fetch(`${dbUrl}/${path}.json`, {
    method: 'PUT',
    body: JSON.stringify(data)
  });
}

async function getData(path) {
  const res = await fetch(`${dbUrl}/${path}.json`);
  return res.json();
}

function generateRoomId() {
  return Math.random().toString(36).substring(2, 8);
}

function QRCode({ url, size = 200 }) {
  const qrUrl = `https://api.qrserver.com/v1/create-qr-code/?size=${size}x${size}&data=${encodeURIComponent(url)}`;
  return <img src={qrUrl} alt="QR Code" style={{width: size, height: size}} />;
}

function formatTime(seconds) {
  if (seconds < 0) seconds = 0;
  const m = Math.floor(seconds / 60);
  const s = seconds % 60;
  return `${m.toString().padStart(2, '0')}:${s.toString().padStart(2, '0')}`;
}

const colors = {
  bluScurissimo: '#00172E',
  bluScuro: '#003569',
  blu: '#028AC1',
  celesteChiaro: '#31BCE3',
  grigio: '#F4F4F4',
  grigioScuro: '#D0D8DD',
  arancio: '#FF7300',
};

function AdminView({ roomId }) {
  const [title, setTitle] = useState('');
  const [minutes, setMinutes] = useState(5);
  const [timerState, setTimerState] = useState(null);
  const [copied, setCopied] = useState(false);
  const pollRef = useRef(null);
  
  const participantUrl = `${window.location.origin}${window.location.pathname}?room=${roomId}`;
  
  useEffect(() => {
    const poll = async () => {
      const data = await getData(`timers/${roomId}`);
      if (data) setTimerState(data);
    };
    poll();
    pollRef.current = setInterval(poll, 1000);
    return () => clearInterval(pollRef.current);
  }, [roomId]);
  
  const startTimer = async () => {
    const now = Date.now();
    const data = {
      title: title || 'Attività',
      duration: minutes * 60,
      remaining: minutes * 60,
      status: 'running',
      startedAt: now,
      pausedAt: null
    };
    await setData(`timers/${roomId}`, data);
    setTimerState(data);
  };
  
  const pauseTimer = async () => {
    if (!timerState) return;
    const elapsed = Math.floor((Date.now() - timerState.startedAt) / 1000);
    const remaining = Math.max(0, timerState.duration - elapsed);
    const data = { ...timerState, status: 'paused', remaining, pausedAt: Date.now() };
    await setData(`timers/${roomId}`, data);
    setTimerState(data);
  };
  
  const resumeTimer = async () => {
    if (!timerState) return;
    const data = { 
      ...timerState, 
      status: 'running', 
      startedAt: Date.now(),
      duration: timerState.remaining,
      pausedAt: null 
    };
    await setData(`timers/${roomId}`, data);
    setTimerState(data);
  };
  
  const stopTimer = async () => {
    if (!timerState) return;
    const data = { ...timerState, status: 'stopped', remaining: 0 };
    await setData(`timers/${roomId}`, data);
    setTimerState(data);
  };
  
  const adjustTime = async (delta) => {
    if (!timerState) return;
    let newRemaining, newDuration;
    
    if (timerState.status === 'running') {
      const elapsed = Math.floor((Date.now() - timerState.startedAt) / 1000);
      const currentRemaining = timerState.duration - elapsed;
      newRemaining = Math.max(0, currentRemaining + delta);
      newDuration = elapsed + newRemaining;
    } else if (timerState.status === 'paused') {
      newRemaining = Math.max(0, timerState.remaining + delta);
      newDuration = newRemaining;
    } else {
      return;
    }
    
    const data = { ...timerState, duration: newDuration, remaining: newRemaining };
    await setData(`timers/${roomId}`, data);
    setTimerState(data);
  };
  
  const copyLink = () => {
    navigator.clipboard.writeText(participantUrl);
    setCopied(true);
    setTimeout(() => setCopied(false), 2000);
  };
  
  const getCurrentRemaining = () => {
    if (!timerState) return 0;
    if (timerState.status === 'stopped') return 0;
    if (timerState.status === 'paused') return timerState.remaining;
    const elapsed = Math.floor((Date.now() - timerState.startedAt) / 1000);
    return Math.max(0, timerState.duration - elapsed);
  };
  
  const isRunning = timerState?.status === 'running';
  const isPaused = timerState?.status === 'paused';
  const isStopped = timerState?.status === 'stopped';
  const isActive = isRunning || isPaused;
  const isTimeUp = isStopped || (timerState && getCurrentRemaining() === 0);
  
  return (
    <div style={{minHeight: '100vh', backgroundColor: colors.bluScurissimo, color: 'white', padding: 20}}>
      <div style={{maxWidth: 600, margin: '0 auto'}}>
        <h1 style={{textAlign: 'center', marginBottom: 30, color: colors.grigio, fontWeight: 300, fontSize: 24}}>
          Workshop Timer
        </h1>
        
        <div style={{backgroundColor: colors.bluScuro, borderRadius: 12, padding: 20, marginBottom: 20}}>
          <h3 style={{margin: '0 0 15px 0', color: colors.celesteChiaro, fontSize: 13, textTransform: 'uppercase', letterSpacing: 1}}>Configurazione</h3>
          <input
            type="text"
            placeholder="Titolo attività"
            value={title}
            onChange={(e) => setTitle(e.target.value)}
            disabled={isActive}
            style={{
              width: '100%', padding: 12, marginBottom: 10, borderRadius: 8,
              border: 'none', fontSize: 16, backgroundColor: colors.bluScurissimo, color: 'white',
              boxSizing: 'border-box'
            }}
          />
          <div style={{display: 'flex', alignItems: 'center', gap: 10, marginBottom: 15}}>
            <span style={{color: colors.grigioScuro}}>Durata:</span>
            <button onClick={() => setMinutes(m => Math.max(1, m - 1))} disabled={isActive}
              style={{padding: '8px 15px', borderRadius: 6, border: 'none', backgroundColor: colors.blu, color: 'white', cursor: 'pointer', opacity: isActive ? 0.5 : 1}}>-</button>
            <span style={{fontSize: 24, minWidth: 70, textAlign: 'center'}}>{minutes} min</span>
            <button onClick={() => setMinutes(m => m + 1)} disabled={isActive}
              style={{padding: '8px 15px', borderRadius: 6, border: 'none', backgroundColor: colors.blu, color: 'white', cursor: 'pointer', opacity: isActive ? 0.5 : 1}}>+</button>
          </div>
          <button onClick={startTimer} disabled={isActive}
            style={{
              width: '100%', padding: 15, borderRadius: 8, border: 'none',
              backgroundColor: isActive ? colors.blu : colors.celesteChiaro, 
              color: isActive ? 'rgba(255,255,255,0.5)' : colors.bluScurissimo,
              fontSize: 16, fontWeight: 600, cursor: isActive ? 'default' : 'pointer',
              transition: 'all 0.2s'
            }}>
            AVVIA TIMER
          </button>
        </div>
        
        {timerState && (
          <div style={{
            backgroundColor: isTimeUp ? colors.arancio : colors.bluScuro,
            borderRadius: 12, padding: 20, marginBottom: 20, textAlign: 'center',
            transition: 'background-color 0.3s'
          }}>
            <div style={{fontSize: 14, color: isTimeUp ? 'rgba(255,255,255,0.8)' : colors.celesteChiaro, marginBottom: 5}}>
              {timerState.title}
            </div>
            <div style={{fontSize: 72, fontWeight: 'bold', fontFamily: 'monospace', color: 'white'}}>
              {formatTime(getCurrentRemaining())}
            </div>
            <div style={{fontSize: 14, color: 'rgba(255,255,255,0.6)'}}>
              {isRunning ? '▶ In corso' : isPaused ? '⏸ In pausa' : '⏹ Terminato'}
            </div>
          </div>
        )}
        
        {isActive && (
          <div style={{backgroundColor: colors.bluScuro, borderRadius: 12, padding: 20, marginBottom: 20}}>
            <h3 style={{margin: '0 0 15px 0', color: colors.celesteChiaro, fontSize: 13, textTransform: 'uppercase', letterSpacing: 1}}>Controlli</h3>
            <div style={{display: 'flex', gap: 10, marginBottom: 15}}>
              {isRunning ? (
                <button onClick={pauseTimer} style={{flex: 1, padding: 15, borderRadius: 8, border: 'none', backgroundColor: colors.arancio, color: 'white', fontSize: 16, fontWeight: 600, cursor: 'pointer'}}>
                  ⏸ PAUSA
                </button>
              ) : (
                <button onClick={resumeTimer} style={{flex: 1, padding: 15, borderRadius: 8, border: 'none', backgroundColor: colors.celesteChiaro, color: colors.bluScurissimo, fontSize: 16, fontWeight: 600, cursor: 'pointer'}}>
                  ▶ RIPRENDI
                </button>
              )}
              <button onClick={stopTimer} style={{flex: 1, padding: 15, borderRadius: 8, border: 'none', backgroundColor: colors.arancio, color: 'white', fontSize: 16, fontWeight: 600, cursor: 'pointer'}}>
                ⏹ STOP
              </button>
            </div>
            <div style={{display: 'flex', gap: 10}}>
              <button onClick={() => adjustTime(-60)} style={{flex: 1, padding: 12, borderRadius: 8, border: 'none', backgroundColor: colors.blu, color: 'white', fontSize: 16, cursor: 'pointer'}}>
                -1 min
              </button>
              <button onClick={() => adjustTime(60)} style={{flex: 1, padding: 12, borderRadius: 8, border: 'none', backgroundColor: colors.blu, color: 'white', fontSize: 16, cursor: 'pointer'}}>
                +1 min
              </button>
            </div>
          </div>
        )}
        
        {isStopped && (
          <div style={{backgroundColor: colors.bluScuro, borderRadius: 12, padding: 20, marginBottom: 20}}>
            <button onClick={() => setTimerState(null)} style={{
              width: '100%', padding: 15, borderRadius: 8, border: 'none',
              backgroundColor: colors.celesteChiaro, color: colors.bluScurissimo, fontSize: 16, fontWeight: 600, cursor: 'pointer'
            }}>
              NUOVO TIMER
            </button>
          </div>
        )}
        
        <div style={{backgroundColor: colors.bluScuro, borderRadius: 12, padding: 20, textAlign: 'center'}}>
          <h3 style={{margin: '0 0 15px 0', color: colors.celesteChiaro, fontSize: 13, textTransform: 'uppercase', letterSpacing: 1}}>Link partecipanti</h3>
          <div style={{backgroundColor: 'white', borderRadius: 8, padding: 15, display: 'inline-block', marginBottom: 15}}>
            <QRCode url={participantUrl} size={180} />
          </div>
          <div style={{fontSize: 12, color: colors.grigioScuro, wordBreak: 'break-all', marginBottom: 10}}>{participantUrl}</div>
          <button onClick={copyLink} style={{
            padding: '10px 20px', borderRadius: 6, border: 'none',
            backgroundColor: colors.blu, color: 'white', cursor: 'pointer', fontWeight: 500
          }}>
            {copied ? '✓ Copiato!' : 'Copia link'}
          </button>
        </div>
      </div>
    </div>
  );
}

function ParticipantView({ roomId }) {
  const [timerState, setTimerState] = useState(null);
  const [remaining, setRemaining] = useState(0);
  const pollRef = useRef(null);
  const tickRef = useRef(null);
  
  useEffect(() => {
    const poll = async () => {
      const data = await getData(`timers/${roomId}`);
      setTimerState(data);
    };
    poll();
    pollRef.current = setInterval(poll, 1000);
    return () => clearInterval(pollRef.current);
  }, [roomId]);
  
  useEffect(() => {
    const tick = () => {
      if (!timerState) return;
      if (timerState.status === 'stopped') {
        setRemaining(0);
      } else if (timerState.status === 'paused') {
        setRemaining(timerState.remaining);
      } else {
        const elapsed = Math.floor((Date.now() - timerState.startedAt) / 1000);
        setRemaining(Math.max(0, timerState.duration - elapsed));
      }
    };
    tick();
    tickRef.current = setInterval(tick, 100);
    return () => clearInterval(tickRef.current);
  }, [timerState]);
  
  const isTimeUp = timerState && (timerState.status === 'stopped' || remaining === 0);
  
  if (!timerState) {
    return (
      <div style={{minHeight: '100vh', backgroundColor: colors.bluScurissimo, color: 'white', display: 'flex', alignItems: 'center', justifyContent: 'center'}}>
        <div style={{textAlign: 'center'}}>
          <div style={{fontSize: 24, marginBottom: 10, color: colors.grigio}}>In attesa...</div>
          <div style={{color: colors.grigioScuro}}>Il timer non è ancora stato avviato</div>
        </div>
      </div>
    );
  }
  
  return (
    <div style={{
      minHeight: '100vh',
      backgroundColor: isTimeUp ? colors.arancio : colors.bluScurissimo,
      color: 'white',
      display: 'flex',
      flexDirection: 'column',
      alignItems: 'center',
      justifyContent: 'center',
      transition: 'background-color 0.3s',
      padding: 20
    }}>
      <div style={{
        fontSize: 'clamp(20px, 5vw, 32px)', 
        marginBottom: 20, 
        textAlign: 'center', 
        color: isTimeUp ? 'rgba(255,255,255,0.9)' : colors.celesteChiaro,
        fontWeight: 300
      }}>
        {timerState.title}
      </div>
      <div style={{
        fontSize: 'clamp(80px, 28vw, 220px)', 
        fontWeight: 'bold', 
        fontFamily: 'monospace', 
        lineHeight: 1,
        color: 'white'
      }}>
        {formatTime(remaining)}
      </div>
      {timerState.status === 'paused' && (
        <div style={{marginTop: 20, fontSize: 18, color: 'rgba(255,255,255,0.7)'}}>⏸ In pausa</div>
      )}
    </div>
  );
}

export default function App() {
  const [view, setView] = useState(null);
  const [roomId, setRoomId] = useState(null);
  
  useEffect(() => {
    const params = new URLSearchParams(window.location.search);
    const room = params.get('room');
    if (room) {
      setRoomId(room);
      setView('participant');
    }
  }, []);
  
  if (view === 'participant' && roomId) {
    return <ParticipantView roomId={roomId} />;
  }
  
  if (view === 'admin' && roomId) {
    return <AdminView roomId={roomId} />;
  }
  
  return (
    <div style={{minHeight: '100vh', backgroundColor: colors.bluScurissimo, color: 'white', display: 'flex', alignItems: 'center', justifyContent: 'center', padding: 20}}>
      <div style={{textAlign: 'center', maxWidth: 400}}>
        <h1 style={{marginBottom: 10, fontWeight: 300, color: colors.grigio}}>Workshop Timer</h1>
        <p style={{color: colors.grigioScuro, marginBottom: 40}}>Timer sincronizzato per facilitatori e partecipanti</p>
        <button 
          onClick={() => { setRoomId(generateRoomId()); setView('admin'); }}
          style={{
            width: '100%', padding: 20, marginBottom: 15, borderRadius: 12, border: 'none',
            backgroundColor: colors.celesteChiaro, color: colors.bluScurissimo, fontSize: 18, fontWeight: 600, cursor: 'pointer'
          }}>
          Crea nuovo timer
        </button>
        <p style={{color: colors.grigioScuro, fontSize: 14}}>I partecipanti accederanno tramite QR code o link</p>
      </div>
    </div>
  );
}
