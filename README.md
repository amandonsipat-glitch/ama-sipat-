import React, { useState, useRef, useEffect } from "react";

// Wingo Aviator - Fun Demo (No real money)
// Single-file React component (Tailwind CSS classes used)
// Features:
// - place virtual bet (coins)
// - start round: multiplier grows from 1.00 upward
// - random "crash" multiplier ends the round
// - user can "cash out" to lock winnings before crash
// - round history shown
// - clearly marked as a demo (no real money)

export default function WingoAviatorFunDemo() {
  const [balance, setBalance] = useState(1000); // virtual coins
  const [bet, setBet] = useState(50);
  const [roundActive, setRoundActive] = useState(false);
  const [multiplier, setMultiplier] = useState(1.0);
  const [crashAt, setCrashAt] = useState(null);
  const [message, setMessage] = useState("");
  const [history, setHistory] = useState([]);
  const [autoCashOut, setAutoCashOut] = useState(0);
  const intervalRef = useRef(null);
  const startTimeRef = useRef(null);

  // Utility: format multiplier
  const fmt = (n) => n.toFixed(2);

  // Start a new round
  const startRound = () => {
    if (roundActive) return;
    if (bet <= 0 || bet > balance) {
      setMessage("Bet must be >0 and <= your balance");
      return;
    }
    // Deduct bet immediately
    setBalance((b) => b - bet);
    setRoundActive(true);
    setMultiplier(1.0);
    setMessage("Take off!");
    // Choose a random crash multiplier between 1.1 and 10 (skewed)
    // use an exponential distribution to make low crashes more common
    const r = Math.random();
    const crash = Math.max(1.02, Math.exp(r * Math.log(10))); // between ~1 and 10
    setCrashAt(crash);
    startTimeRef.current = Date.now();
    // Increase multiplier over time
    intervalRef.current = setInterval(() => {
      setMultiplier((m) => {
        const next = m * 1.003; // smooth growth
        // If passes crash, finish round
        if (next >= crash) {
          finishRound(false, crash);
          return crash; // final
        }
        // If auto cashout set and reached, cash out
        if (autoCashOut > 0 && next >= autoCashOut) {
          finishRound(true, next);
          return next;
        }
        return next;
      });
    }, 40); // 25 updates/sec
  };

  // Cash out manually
  const cashOut = () => {
    if (!roundActive) return;
    finishRound(true, multiplier);
  };

  // Finish the round: success=true if cashed out before crash
  const finishRound = (success, finalMultiplier) => {
    if (!roundActive) return;
    clearInterval(intervalRef.current);
    intervalRef.current = null;
    setRoundActive(false);

    const round = {
      time: new Date().toLocaleTimeString(),
      bet,
      crashedAt: crashAt ? fmt(crashAt) : "-",
      cashedOutAt: success ? fmt(finalMultiplier) : "CRASH",
      result: success ? "WIN" : "LOSS",
      payout: success ? +(bet * finalMultiplier).toFixed(2) : 0,
    };

    if (success) {
      const payout = +(bet * finalMultiplier).toFixed(2);
      setBalance((b) => +(b + payout).toFixed(2));
      setMessage(`Cashed out Ã—${fmt(finalMultiplier)} â€” +${payout} coins`);
    } else {
      setMessage(`Crashed at Ã—${fmt(finalMultiplier)} â€” You lost ${bet} coins`);
    }

    setHistory((h) => [round, ...h].slice(0, 12));
    setMultiplier(1.0);
    setCrashAt(null);
    startTimeRef.current = null;
  };

  useEffect(() => {
    return () => {
      if (intervalRef.current) clearInterval(intervalRef.current);
    };
  }, []);

  // Quick controls to adjust bet
  const adjustBet = (val) => {
    setBet((prev) => {
      const next = Math.round((prev + val) * 100) / 100;
      if (next < 1) return 1;
      if (next > balance) return balance;
      return next;
    });
  };

  return (
    <div className="min-h-screen bg-gradient-to-b from-sky-50 to-white p-6 flex flex-col items-center">
      <div className="w-full max-w-3xl">
        <header className="flex items-center justify-between mb-6">
          <div>
            <h1 className="text-3xl font-extrabold">Wingo Aviator â€” Fun Demo</h1>
            <p className="text-sm text-gray-600">Demo mode â€” no real money. Practice and have fun!</p>
          </div>
          <div className="text-right">
            <div className="text-sm text-gray-500">Balance</div>
            <div className="text-xl font-semibold">{balance.toFixed(2)} coins</div>
          </div>
        </header>

        <main className="bg-white rounded-2xl shadow p-6">
          <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
            {/* Left: game board */}
            <div className="col-span-2">
              <div className="flex items-center justify-between mb-3">
                <div>
                  <div className="text-xs text-gray-500">Multiplier</div>
                  <div className="text-4xl font-bold">Ã— {fmt(multiplier)}</div>
                </div>
                <div className="text-right">
                  <div className="text-xs text-gray-500">Status</div>
                  <div className={`font-semibold ${roundActive ? 'text-green-600' : 'text-gray-700'}`}>
                    {roundActive ? 'In Flight' : 'Waiting'}
                  </div>
                </div>
              </div>

              <div className="h-48 rounded-lg bg-gradient-to-r from-sky-100 to-sky-50 border border-sky-200 flex items-center justify-center mb-4">
                {/* Simple visual: increasing bar */}
                <div className="w-full px-6">
                  <div className="relative h-6 bg-white/60 rounded-full overflow-hidden border border-sky-200">
                    <div
                      className="absolute left-0 top-0 bottom-0 transition-all"
                      style={{ width: `${Math.min(100, (multiplier / Math.max(10, crashAt || 10)) * 100)}%` }}
                    />
                  </div>
                  <div className="mt-3 text-xs text-gray-600">Crash at: {crashAt ? fmt(crashAt) : 'â€”'}</div>
                </div>
              </div>

              <div className="flex gap-3">
                <input
                  type="number"
                  className="p-3 border rounded w-full"
                  value={bet}
                  onChange={(e) => setBet(Math.max(1, Math.min(balance, +e.target.value || 0)))}
                  disabled={roundActive}
                />
                <div className="flex flex-col gap-2">
                  <button className="px-4 py-2 bg-sky-600 text-white rounded" onClick={() => adjustBet(-10)} disabled={roundActive}>-10</button>
                  <button className="px-4 py-2 bg-sky-600 text-white rounded" onClick={() => adjustBet(10)} disabled={roundActive}>+10</button>
                </div>
              </div>

              <div className="flex gap-3 mt-4">
                <button
                  className={`flex-1 px-4 py-3 rounded font-semibold ${roundActive ? 'bg-gray-300 text-gray-700' : 'bg-green-500 text-white'}`}
                  onClick={startRound}
                  disabled={roundActive}
                >
                  Start Round
                </button>
                <button
                  className={`flex-1 px-4 py-3 rounded font-semibold ${roundActive ? 'bg-orange-500 text-white' : 'bg-gray-200 text-gray-700'}`}
                  onClick={cashOut}
                  disabled={!roundActive}
                >
                  Cash Out
                </button>
              </div>

              <div className="mt-3 text-sm text-gray-600">Auto cashout (Ã—):
                <input
                  type="number"
                  step="0.1"
                  min="1.01"
                  className="ml-2 w-24 p-1 border rounded"
                  value={autoCashOut}
                  onChange={(e) => setAutoCashOut(Math.max(0, +e.target.value || 0))}
                />
                <span className="ml-2 text-xs text-gray-500">Set to 0 to disable</span>
              </div>

              <div className="mt-4 p-3 rounded border border-dashed">
                <div className="text-sm text-gray-700">{message}</div>
              </div>
            </div>

            {/* Right: history & tips */}
            <aside className="col-span-1">
              <div className="mb-4">
                <h2 className="font-semibold">Round History</h2>
                <div className="mt-2 space-y-2 max-h-48 overflow-auto">
                  {history.length === 0 && <div className="text-xs text-gray-500">No rounds yet</div>}
                  {history.map((r, i) => (
                    <div key={i} className="flex items-center justify-between text-sm bg-sky-50 p-2 rounded">
                      <div>
                        <div className="font-semibold">{r.result}</div>
                        <div className="text-xs text-gray-600">Bet {r.bet} â€¢ {r.time}</div>
                      </div>
                      <div className="text-right">
                        <div className="text-sm">{r.cashedOutAt}</div>
                        <div className="text-xs text-gray-600">Crash {r.crashedAt}</div>
                      </div>
                    </div>
                  ))}
                </div>
              </div>

              <div className="p-3 bg-amber-50 rounded">
                <div className="text-xs text-gray-700 font-semibold mb-2">Tips</div>
                <ul className="text-xs text-gray-600 list-disc pl-4 space-y-1">
                  <li>Ye sirf demo hai â€” asli paise nahi.</li>
                  <li>Small bets se practice karo.</li>
                  <li>Auto cashout use karne se discipline aata hai.</li>
                </ul>
              </div>
            </aside>
          </div>
        </main>

        <footer className="mt-6 text-center text-xs text-gray-500">
          Built for fun â€” Wingo Aviator Demo â€¢ No real-money gambling supported
        </footer>
      </div>
    </div>
  );
}
