# Vylert.github.io

import React, { useState, useEffect, useCallback, useMemo } from 'react';
import {
  Zap, Users, TrendingUp, Search, MessageSquare, Menu, X, Copy,
  ArrowRight, Activity, Globe, DollarSign, Loader2, Info, Check, Link // Added Link icon
} from 'lucide-react';

// --- MOCK DATA UTILITIES (Kept for dashboard metrics) ---
const generateMockData = () => {
  const trendingTopics = [
    'Quantum Computing Hype', 'AI Ethics Debate', 'Decentralized Finance Regulation',
    'Sustainable Tech Investments', 'Metaverse Market Correction', 'Cybersecurity Vulnerabilities'
  ];
  const influencerNames = [
    'Sentinel_Zero', 'AetherAnalyst', 'CypherQueen', 'DataDragon', 'TheGridRunner', 'NeonNomad'
  ];
  const randomRange = (min, max) => Math.floor(Math.random() * (max - min + 1)) + min;

  const liveWireData = trendingTopics.map(topic => ({
    id: crypto.randomUUID(),
    topic: topic,
    volume: randomRange(5000, 25000).toLocaleString(),
    sentiment: Math.random() > 0.6 ? 'Positive' : (Math.random() > 0.3 ? 'Neutral' : 'Negative'),
    velocity: randomRange(5, 50) / 10,
  }));

  const radarData = influencerNames.map(name => ({
    id: crypto.randomUUID(),
    name: name,
    score: randomRange(75, 99),
    followers: (randomRange(10, 500) * 1000).toLocaleString(),
    recentActivity: `${randomRange(1, 12)} engagements in the last hour`,
    status: Math.random() > 0.7 ? 'Active' : 'Dormant',
    avatarUrl: `https://placehold.co/40x40/0E7490/FFFFFF?text=${name.charAt(0)}`
  }));

  const globalMetrics = [
    { label: 'Total Volume Scanned', icon: Globe, value: (randomRange(10, 500) * 100000).toLocaleString(), unit: 'Data Pts' },
    { label: 'Active Agents Deployed', icon: Users, value: randomRange(1000, 5000).toLocaleString(), unit: 'Agents' },
    { label: 'Market Volatility Index', icon: Activity, value: (Math.random() * 25).toFixed(2), unit: '%' },
    { label: 'Average Sentiment Score', icon: TrendingUp, value: (Math.random() * 0.5 + 0.5).toFixed(3), unit: '' },
  ];

  return { liveWireData, radarData, globalMetrics };
};

// --- API UTILITIES (New for Gemini Integration) ---

/**
 * Calls the Gemini API with exponential backoff for resilience.
 * Uses Google Search grounding to provide real-time, factual answers.
 */
const callGeminiApi = async (userQuery, systemPrompt, maxRetries = 5) => {
  // NOTE: In the Canvas environment, the API key is handled automatically.
  // We keep it as an empty string here as required by the environment rules.
  const apiKey = ""; 
  const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`;

  const payload = {
    contents: [{ parts: [{ text: userQuery }] }],
    // Enable Google Search for real-time grounding
    tools: [{ "google_search": {} }],
    systemInstruction: { parts: [{ text: systemPrompt }] },
  };

  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      const response = await fetch(apiUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });

      if (response.status === 429 && attempt < maxRetries - 1) {
        // Exponential backoff
        const delay = Math.pow(2, attempt) * 1000 + Math.random() * 1000;
        await new Promise(resolve => setTimeout(resolve, delay));
        continue; // Retry
      }
      
      if (!response.ok) {
        throw new Error(`API request failed with status: ${response.status}`);
      }

      const result = await response.json();
      const candidate = result.candidates?.[0];

      if (candidate && candidate.content?.parts?.[0]?.text) {
        const text = candidate.content.parts[0].text;
        let sources = [];
        const groundingMetadata = candidate.groundingMetadata;

        if (groundingMetadata && groundingMetadata.groundingAttributions) {
            sources = groundingMetadata.groundingAttributions
                .map(attr => ({
                    uri: attr.web?.uri,
                    title: attr.web?.title,
                }))
                .filter(source => source.uri && source.title);
        }
        return { text, sources };
      }
      return { text: "Error: The model returned an invalid response structure.", sources: [] };

    } catch (error) {
      console.error(`Attempt ${attempt + 1} failed:`, error.message);
      if (attempt === maxRetries - 1) {
        return { text: `System Error: Failed to connect to the data synthesis engine after ${maxRetries} attempts. Please check network or try again later.`, sources: [] };
      }
      const delay = Math.pow(2, attempt) * 1000 + Math.random() * 1000;
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
};


// --- UI COMPONENTS ---

// 1. Vylert Logo Component
const VylertLogo = () => (
  <div className="flex items-center space-x-2 p-4">
    <Zap className="w-8 h-8 text-cyan-400 drop-shadow-[0_0_5px_rgba(45,212,255,0.8)]" />
    <h1 className="text-3xl font-extrabold tracking-widest text-white uppercase font-sans">
      <span className="text-cyan-400">VY</span>LERT
    </h1>
    <span className="text-xs text-indigo-400 ml-1 font-mono tracking-wider">2.0</span>
  </div>
);

// 2. Copy To Clipboard Component
const CopyToClipboard = ({ textToCopy, className = '' }) => {
  const [copied, setCopied] = useState(false);

  const handleCopy = () => {
    const textarea = document.createElement('textarea');
    textarea.value = textToCopy;
    textarea.style.position = 'fixed';
    document.body.appendChild(textarea);
    textarea.focus();
    textarea.select();
    try {
      // Use document.execCommand('copy') as navigator.clipboard may not work in iframes
      document.execCommand('copy');
      setCopied(true);
      setTimeout(() => setCopied(false), 2000);
    } catch (err) {
      console.error('Failed to copy text: ', err);
    }
    document.body.removeChild(textarea);
  };

  return (
    <button
      onClick={handleCopy}
      className={`p-1 rounded-full transition duration-300 ${copied ? 'bg-emerald-600 text-white' : 'hover:bg-cyan-700/50 text-cyan-400'} ${className}`}
      title={copied ? 'Copied!' : 'Copy to Clipboard'}
    >
      {copied ? <Check className="w-4 h-4" /> : <Copy className="w-4 h-4" />}
    </button>
  );
};


// 3. Live Wire Scanner Component
const LiveWireScanner = ({ data }) => {
  const getSentimentClass = (sentiment) => {
    switch (sentiment) {
      case 'Positive': return 'text-emerald-400 bg-emerald-900/50';
      case 'Negative': return 'text-rose-400 bg-rose-900/50';
      default: return 'text-yellow-400 bg-yellow-900/50';
    }
  };
  const getVelocityIcon = (velocity) => {
    if (velocity >= 4.0) return '🚀';
    if (velocity >= 2.0) return '🔥';
    return '⚡';
  };

  return (
    <div className="h-full flex flex-col">
      <h2 className="text-xl font-bold text-white mb-4 flex items-center">
        <Activity className="w-5 h-5 mr-2 text-red-500" />
        LiveWire Scanner: High-Velocity Topics
      </h2>
      <div className="overflow-y-auto custom-scrollbar flex-grow space-y-3 pr-2">
        {data.map((item, index) => (
          <div
            key={item.id}
            className={`
              p-4 rounded-xl border border-cyan-800/50 backdrop-blur-sm
              hover:border-cyan-500/80 transition duration-300 ease-in-out
              bg-gray-900/40 hover:bg-gray-800/60
              shadow-lg shadow-black/30
            `}
            style={{ animationDelay: `${index * 0.1}s` }}
          >
            <div className="flex justify-between items-center mb-2">
              <span className="text-sm font-semibold text-white truncate">
                {getVelocityIcon(item.velocity)} {item.topic}
              </span>
              <span className={`px-2 py-0.5 text-xs font-medium rounded-full ${getSentimentClass(item.sentiment)}`}>
                {item.sentiment}
              </span>
            </div>
            <div className="flex justify-between text-xs text-gray-400 font-mono">
              <span>Volume: <span className="text-cyan-300">{item.volume}</span></span>
              <span>Velocity: <span className="text-red-400">{item.velocity}x</span></span>
            </div>
          </div>
        ))}
      </div>
    </div>
  );
};


// 4. Influencer Radar Component
const InfluencerRadar = ({ data }) => {
  const [hoveredInfluencer, setHoveredInfluencer] = useState(null);

  const InfluencerProfileOverlay = ({ influencer }) => (
    <div className="absolute z-50 top-0 left-full ml-4 p-4 w-64 bg-gray-800/95 backdrop-blur-md rounded-xl border border-purple-600 shadow-xl shadow-purple-900/50 transform translate-x-1">
      <h4 className="font-bold text-lg text-purple-400">{influencer.name}</h4>
      <p className="text-sm text-gray-300 mb-2">Influence Score: <span className="text-yellow-400">{influencer.score}</span></p>
      <ul className="text-xs space-y-1 text-gray-400">
        <li>Followers: <span className="font-mono text-white">{influencer.followers}</span></li>
        <li>Status: <span className={`font-mono ${influencer.status === 'Active' ? 'text-emerald-400' : 'text-orange-400'}`}>{influencer.status}</span></li>
        <li>Activity: {influencer.recentActivity}</li>
      </ul>
      <button className="mt-3 w-full text-center bg-purple-700 hover:bg-purple-600 text-white text-xs font-medium py-1 rounded-lg transition duration-200 flex items-center justify-center">
        Analyze Deep Dive <ArrowRight className="w-3 h-3 ml-1" />
      </button>
    </div>
  );

  return (
    <div className="h-full flex flex-col">
      <h2 className="text-xl font-bold text-white mb-4 flex items-center">
        <Users className="w-5 h-5 mr-2 text-purple-500" />
        Influencer Radar: Key Agents
      </h2>
      <div className="overflow-y-auto custom-scrollbar flex-grow pr-2">
        <ul className="space-y-3">
          {data
            .sort((a, b) => b.score - a.score)
            .map((item, index) => (
              <li
                key={item.id}
                className="relative p-3 bg-gray-900/40 border-l-4 border-purple-500/70 rounded-lg hover:bg-gray-800/60 transition duration-300"
                onMouseEnter={() => setHoveredInfluencer(item)}
                onMouseLeave={() => setHoveredInfluencer(null)}
              >
                <div className="flex items-center justify-between">
                  <div className="flex items-center">
                    <img
                      src={item.avatarUrl}
                      alt={item.name}
                      className="w-8 h-8 rounded-full border-2 border-purple-400 mr-3"
                    />
                    <span className="text-sm font-medium text-white">{item.name}</span>
                  </div>
                  <div className="flex items-center space-x-2">
                    <span className={`text-xs font-mono font-bold ${item.score > 90 ? 'text-yellow-300' : 'text-purple-300'}`}>
                      {item.score}
                    </span>
                    <span className="text-xs text-gray-500">
                      {item.status === 'Active' ? <Zap className="w-4 h-4 text-emerald-400" title="Active" /> : <Loader2 className="w-4 h-4 text-orange-400 animate-spin" title="Dormant" />}
                    </span>
                  </div>
                </div>
                {hoveredInfluencer && hoveredInfluencer.id === item.id && (
                  <InfluencerProfileOverlay influencer={hoveredInfluencer} />
                )}
              </li>
            ))}
        </ul>
      </div>
    </div>
  );
};


// 5. Vylert AI Chat Component (Updated for Gemini API Integration)
const VylertAIChat = () => {
  const [messages, setMessages] = useState([
    { id: 1, text: "Vylert AI: Protocol: Online. I am connected to the Global Nexus and can provide real-time, grounded data synthesis. Ask me anything about the market or trending topics.", sender: 'ai', sources: [] },
  ]);
  const [input, setInput] = useState('');
  const [isTyping, setIsTyping] = useState(false);

  const systemPrompt = "You are Vylert AI, a highly specialized, concise, and professional data synthesis engine. Your goal is to provide brief, factual, and actionable summaries based on real-time data. Always prioritize information from the provided sources. If a user asks a simple greeting, respond with a professional status update. Do not elaborate unless prompted.";

  const handleSend = async (e) => {
    e.preventDefault();
    if (input.trim() === '' || isTyping) return;

    const userQuery = input.trim();
    const newMessage = { id: Date.now(), text: userQuery, sender: 'user', sources: [] };
    setMessages((prev) => [...prev, newMessage]);
    setInput('');
    setIsTyping(true);

    try {
      const response = await callGeminiApi(userQuery, systemPrompt);
      const aiResponse = { id: Date.now() + 1, text: response.text, sender: 'ai', sources: response.sources };
      setMessages((prev) => [...prev, aiResponse]);
    } catch (error) {
      console.error("Gemini API call failed:", error);
      const errorResponse = { id: Date.now() + 1, text: "Error: Could not process the request. Check the console for details.", sender: 'ai', sources: [] };
      setMessages((prev) => [...prev, errorResponse]);
    } finally {
      setIsTyping(false);
    }
  };

  const ChatMessage = ({ msg }) => (
    <div
        key={msg.id}
        className={`flex ${msg.sender === 'user' ? 'justify-end' : 'justify-start'}`}
    >
        <div
            className={`max-w-[85%] p-3 rounded-xl shadow-lg
            ${msg.sender === 'user'
                ? 'bg-purple-700/70 text-white rounded-br-none'
                : 'bg-gray-800/80 text-gray-200 rounded-tl-none border border-cyan-700/50'
            }
            transition-all duration-300 ease-out`}
        >
            <p className="text-sm">{msg.text}</p>
            {msg.sources && msg.sources.length > 0 && (
                <div className="mt-2 pt-2 border-t border-gray-700/50">
                    <p className="text-xs font-semibold text-cyan-400 mb-1 flex items-center">
                        <Link className="w-3 h-3 mr-1" /> Grounding Sources:
                    </p>
                    <ul className="space-y-1 text-xs text-gray-400 list-none pl-0">
                        {msg.sources.slice(0, 3).map((source, index) => (
                            <li key={index} className="truncate hover:text-cyan-300">
                                <a href={source.uri} target="_blank" rel="noopener noreferrer" title={source.title} className="flex items-center space-x-1">
                                    <span className="text-purple-400">[{index + 1}]</span>
                                    <span className="flex-1 truncate">{source.title}</span>
                                </a>
                            </li>
                        ))}
                    </ul>
                </div>
            )}
        </div>
    </div>
  );

  return (
    <div className="h-full flex flex-col p-4">
      <h2 className="text-xl font-bold text-white mb-4 flex items-center">
        <MessageSquare className="w-5 h-5 mr-2 text-emerald-400" />
        Vylert AI Assistant
      </h2>
      <div className="flex-grow overflow-y-auto custom-scrollbar pr-2 space-y-4 mb-4">
        {messages.map((msg) => <ChatMessage key={msg.id} msg={msg} />)}
        {isTyping && (
          <div className="flex justify-start">
            <div className="p-3 rounded-xl bg-gray-800/80 text-gray-400 text-sm italic">
              Vylert AI is synthesizing data...
              <Loader2 className="w-4 h-4 inline ml-2 animate-spin text-cyan-400" />
            </div>
          </div>
        )}
      </div>
      <form onSubmit={handleSend} className="flex space-x-2 pt-2 border-t border-cyan-800">
        <input
          type="text"
          value={input}
          onChange={(e) => setInput(e.target.value)}
          placeholder="Ask the AI for real-time market synthesis..."
          className="flex-grow p-3 bg-gray-900/80 border border-cyan-700/50 rounded-xl text-white placeholder-gray-500 focus:ring-2 focus:ring-cyan-500 outline-none transition duration-200"
          disabled={isTyping}
        />
        <button
          type="submit"
          className="bg-cyan-600 hover:bg-cyan-500 p-3 rounded-xl text-white font-bold transition duration-200 shadow-md shadow-cyan-900/50 disabled:bg-gray-700 disabled:cursor-not-allowed"
          disabled={isTyping || input.trim() === ''}
        >
          <ArrowRight className="w-5 h-5" />
        </button>
      </form>
    </div>
  );
};


// 6. Global Metrics Card Component
const GlobalMetricsCard = ({ metric }) => {
  const Icon = metric.icon;

  const colorClass = useMemo(() => {
    switch (metric.label) {
      case 'Total Volume Scanned': return 'text-cyan-400 border-cyan-600';
      case 'Active Agents Deployed': return 'text-purple-400 border-purple-600';
      case 'Market Volatility Index': return 'text-red-400 border-red-600';
      case 'Average Sentiment Score': return 'text-emerald-400 border-emerald-600';
      default: return 'text-gray-400 border-gray-600';
    }
  }, [metric.label]);

  return (
    <div className={`p-5 rounded-2xl bg-gray-900/40 border-l-4 ${colorClass} shadow-xl shadow-black/50 transition duration-300 hover:scale-[1.02] cursor-pointer`}>
      <div className="flex items-center justify-between">
        <p className="text-sm font-medium text-gray-400 uppercase tracking-widest">{metric.label}</p>
        <Icon className={`w-6 h-6 ${colorClass}`} />
      </div>
      <div className="mt-3">
        <p className="text-3xl font-extrabold text-white">
          {metric.value}
          <span className="text-sm ml-1 font-mono text-gray-500">{metric.unit}</span>
        </p>
      </div>
      <div className="text-xs mt-2 font-mono text-gray-500">
        <span className="text-emerald-400">+2.5%</span> last 24h
      </div>
    </div>
  );
};

// 7. NavItem Component
const NavItem = ({ Icon, label, isActive }) => (
  <a href="#" className={`flex items-center p-3 rounded-xl transition duration-300 ${isActive ? 'bg-cyan-800/50 text-white border-l-4 border-cyan-400' : 'text-gray-400 hover:bg-gray-800/50 hover:text-cyan-400'}`}>
    <Icon className="w-5 h-5 mr-3" />
    <span className="font-medium">{label}</span>
  </a>
);

// 8. Header Component
const Header = ({ isSidebarOpen, setIsSidebarOpen }) => (
  <header className="flex justify-between items-center p-4 bg-gray-900/60 backdrop-blur-md border-b border-cyan-900/50 shadow-lg shadow-black/30 sticky top-0 z-10">
    <div className="flex items-center space-x-4">
      <button
        className="lg:hidden text-cyan-400"
        onClick={() => setIsSidebarOpen(!isSidebarOpen)}
      >
        {isSidebarOpen ? <X className="w-6 h-6" /> : <Menu className="w-6 h-6" />}
      </button>
      <div className="hidden sm:block">
        <VylertLogo />
      </div>
      <div className="relative hidden md:block">
        <Search className="w-5 h-5 absolute top-1/2 -translate-y-1/2 left-3 text-gray-500" />
        <input
          type="text"
          placeholder="Search Agents, Keywords, or Artifacts..."
          className="w-80 p-2 pl-10 bg-gray-800/80 border border-cyan-900 rounded-full text-white text-sm focus:ring-2 focus:ring-cyan-500 outline-none"
        />
      </div>
    </div>
    <div className="flex items-center space-x-3">
      <Info className="w-5 h-5 text-gray-400 hover:text-cyan-400 cursor-pointer transition" title="System Status: Optimal" />
      <div className="text-sm font-mono text-gray-400 hidden sm:block">
        USER_ID: <span className="text-purple-400">#AETHER-8921</span>
      </div>
      <div className="w-8 h-8 rounded-full bg-purple-600 flex items-center justify-center text-white font-bold cursor-pointer hover:bg-purple-500 transition">
        S
      </div>
    </div>
  </header>
);

// 9. Sidebar Component
const Sidebar = ({ isSidebarOpen, setIsSidebarOpen }) => (
  <div className={`fixed inset-y-0 left-0 z-40 lg:relative lg:translate-x-0 transform ${isSidebarOpen ? 'translate-x-0' : '-translate-x-full'} w-64 bg-gray-900 border-r border-cyan-900/50 transition-transform duration-300 ease-in-out flex flex-col p-4`}>
    <div className="mb-8 flex justify-between items-center">
      <VylertLogo />
      <button className="lg:hidden text-cyan-400 p-2" onClick={() => setIsSidebarOpen(false)}>
        <X className="w-6 h-6" />
      </button>
    </div>
    <nav className="space-y-2 flex-grow">
      <NavItem Icon={Zap} label="Dashboard" isActive={true} />
      <NavItem Icon={Users} label="Agent Network" isActive={false} />
      <NavItem Icon={TrendingUp} label="Market Synthesis" isActive={false} />
      <NavItem Icon={DollarSign} label="Financial Projections" isActive={false} />
      <NavItem Icon={Globe} label="Global Nexus" isActive={false} />
    </nav>
    <div className="mt-auto pt-4 border-t border-cyan-900/50">
      <p className="text-xs text-gray-500 font-mono">
        Vylert OS v2.0.1 | Status: <span className="text-emerald-400">Online</span>
      </p>
    </div>
  </div>
);


// 10. Main App Component
const App = () => {
  const [data, setData] = useState(generateMockData());
  const [isSidebarOpen, setIsSidebarOpen] = useState(false);

  // Simulate real-time data updates for dashboard metrics every 5 seconds
  useEffect(() => {
    const interval = setInterval(() => {
      setData(generateMockData());
    }, 5000);
    return () => clearInterval(interval);
  }, []);


  return (
    <div className="min-h-screen bg-gray-950 text-white font-sans flex antialiased">
      <Sidebar isSidebarOpen={isSidebarOpen} setIsSidebarOpen={setIsSidebarOpen} />

      <main className="flex-grow flex flex-col overflow-x-hidden">
        <Header isSidebarOpen={isSidebarOpen} setIsSidebarOpen={setIsSidebarOpen} />

        <div className="p-4 sm:p-6 lg:p-8 flex-grow">
          <h1 className="text-4xl font-extrabold text-white mb-6 tracking-tight">
            Operational Dashboard
          </h1>

          {/* Global Metrics Grid */}
          <div className="grid grid-cols-1 md:grid-cols-2 xl:grid-cols-4 gap-6 mb-8">
            {data.globalMetrics.map((metric) => (
              <GlobalMetricsCard key={metric.label} metric={metric} />
            ))}
          </div>

          {/* Main Content Panels */}
          <div className="grid grid-cols-1 lg:grid-cols-3 gap-6 h-[70vh] min-h-[500px]">
            {/* Panel 1: Live Wire Scanner */}
            <div className="lg:col-span-1 p-6 bg-gray-900/50 border border-cyan-800/40 rounded-3xl shadow-2xl shadow-black/50 overflow-hidden">
              <LiveWireScanner data={data.liveWireData} />
            </div>

            {/* Panel 2: Influencer Radar */}
            <div className="lg:col-span-1 p-6 bg-gray-900/50 border border-purple-800/40 rounded-3xl shadow-2xl shadow-black/50 overflow-hidden">
              <InfluencerRadar data={data.radarData} />
            </div>

            {/* Panel 3: Vylert AI Assistant (Now powered by Gemini) */}
            <div className="lg:col-span-1 p-6 bg-gray-900/50 border border-emerald-800/40 rounded-3xl shadow-2xl shadow-black/50 overflow-hidden">
              <VylertAIChat />
            </div>
          </div>
        </div>
      </main>

      {/* Custom Scrollbar Style for the Dashboard */}
      <style>{`
        .custom-scrollbar::-webkit-scrollbar {
          width: 8px;
        }
        .custom-scrollbar::-webkit-scrollbar-track {
          background: #0f172a; /* Slate 900 */
        }
        .custom-scrollbar::-webkit-scrollbar-thumb {
          background-color: #06b6d4; /* Cyan 500 */
          border-radius: 20px;
          border: 2px solid #0f172a;
        }
        .custom-scrollbar {
          scrollbar-width: thin;
          scrollbar-color: #06b6d4 #0f172a;
        }
      `}</style>
    </div>
  );
};

export default App;
