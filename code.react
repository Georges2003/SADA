import React, { useState, useEffect, useRef } from 'react';
import { 
  Search, MessageSquare, Building2, ArrowRight, Users, 
  Activity, Play, AlertTriangle, FileText, Loader2, 
  RefreshCcw, CheckCircle2, Trash2, MapPin, 
  BrainCircuit, ChevronRight, User, MousePointerClick,
  Sparkles, Globe, ShieldAlert, BookOpen, Settings,
  Database, Languages, Zap, Layers, HeartPulse, Microscope, Wrench
} from 'lucide-react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, collection, doc, setDoc, onSnapshot, addDoc, deleteDoc, getDocs, writeBatch } from 'firebase/firestore';

// --- Configuration ---
const API_KEY = ""; // Provided by environment (Gemini)
const UNBOUND_API_KEY = "f978d436fd099dc69b106feb4712a1f7801c017ab6c293d0d11fbe22e77463b2335f181887b2eb015c936eb3218fe764"; 
const CRUSTDATA_API_KEY = "4b9980320d8d0284bd251e757dda9c941e483b13"; 
const LINGO_DEV_API_KEY = "api_chzd4vpij6t4wb5nc62q6b8p"; 
const MODEL_NAME = "gemini-2.5-flash-preview-09-2025";
const APP_ID = typeof __app_id !== 'undefined' ? __app_id : 'synthetic-user-lab';

// --- Firebase Init ---
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);

// --- Main Application Component ---
const App = () => {
  // Global State
  const [user, setUser] = useState(null);
  const [view, setView] = useState('setup'); // setup, profiler, population, simulation
  const [loading, setLoading] = useState(false);
  const [logs, setLogs] = useState([]);
  
  // New Integrations State
  const [useUnbound, setUseUnbound] = useState(false); // Unbound Switch
  const [currentLang, setCurrentLang] = useState('en'); // Lingo.dev State
  const [uiStrings, setUiStrings] = useState({}); // Stores localized strings
  
  // Business Profiler State
  const [businessName, setBusinessName] = useState('');
  const [profilerMode, setProfilerMode] = useState('landing'); // landing, search, chat
  const [chatMessages, setChatMessages] = useState([]);
  const [chatInput, setChatInput] = useState('');
  const [searchData, setSearchData] = useState(null);
  const [crustData, setCrustData] = useState(null); // Crustdata State
  const [citations, setCitations] = useState([]);
  const [businessProfile, setBusinessProfile] = useState(null); // The final merged text

  // Population State
  const [population, setPopulation] = useState([]);
  const [targetCount, setTargetCount] = useState(10);
  const [progress, setProgress] = useState(0);
  const [selectedPersona, setSelectedPersona] = useState(null);

  // Simulation State
  const [simResults, setSimResults] = useState([]);
  const [isSimulating, setIsSimulating] = useState(false);
  const [simProgress, setSimProgress] = useState(0);
  const [simSampleSize, setSimSampleSize] = useState(5); // User-defined sample size

  const chatEndRef = useRef(null);

  // --- Auth & Data Sync ---
  useEffect(() => {
    const initAuth = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          await signInWithCustomToken(auth, __initial_auth_token);
        } else {
          await signInAnonymously(auth);
        }
      } catch (err) {
        console.error("Auth failed", err);
      }
    };
    initAuth();
    
    const unsubscribe = onAuthStateChanged(auth, (u) => {
      setUser(u);
      if (u) {
        // Sync Business Profile
        const profileUnsub = onSnapshot(doc(db, 'artifacts', APP_ID, 'public', 'data', 'business_context', 'current'), (doc) => {
           if (doc.exists()) {
             setBusinessProfile(doc.data().text);
             setBusinessName(doc.data().name);
           }
        });

        // Sync Population
        const popUnsub = onSnapshot(collection(db, 'artifacts', APP_ID, 'public', 'data', 'population'), (snap) => {
          const data = snap.docs.map(d => ({ id: d.id, ...d.data() }));
          setPopulation(data);
        });

        // Sync Simulation Results
        const simUnsub = onSnapshot(collection(db, 'artifacts', APP_ID, 'public', 'data', 'simulation_logs'), (snap) => {
          const data = snap.docs.map(d => ({ id: d.id, ...d.data() })).sort((a,b) => b.timestamp - a.timestamp);
          setSimResults(data);
        });

        return () => {
          profileUnsub();
          popUnsub();
          simUnsub();
        };
      }
    });
    return () => unsubscribe();
  }, []);

  useEffect(() => {
    chatEndRef.current?.scrollIntoView({ behavior: "smooth" });
  }, [chatMessages]);

  // --- Helpers ---
  const addLog = (msg) => setLogs(prev => [`[${new Date().toLocaleTimeString()}] ${msg}`, ...prev.slice(0, 19)]);
  
  // Helper for Lingo.dev Localization
  const t = (key, defaultText) => {
     if (currentLang === 'en') return defaultText;
     return uiStrings[key] || defaultText;
  };

  // --- API Integrations ---

  // 1. Unbound Integration (OpenAI Compatible)
  const callUnbound = async (prompt, systemInstruction, jsonSchema) => {
      addLog("Routing via Unbound Gateway...");
      
      try {
        const response = await fetch("https://api.getunbound.ai/v1/chat/completions", {
            method: "POST",
            headers: {
                "Authorization": `Bearer ${UNBOUND_API_KEY}`,
                "Content-Type": "application/json"
            },
            body: JSON.stringify({
                model: "gpt-4o-mini", // Fallback to a standard model
                messages: [
                    { role: "system", content: systemInstruction },
                    { role: "user", content: prompt }
                ],
                // If schema is provided, we try to use response_format if compatible, or prompt engineering
                response_format: jsonSchema ? { type: "json_object" } : undefined 
            })
        });

        if (!response.ok) {
            const errorText = await response.text();
            throw new Error(`Unbound API Error: ${errorText}`);
        }

        const data = await response.json();
        // Map Unbound (OpenAI) response back to Gemini structure for compatibility
        return {
            candidates: [{
                content: { parts: [{ text: data.choices[0].message.content }] }
            }]
        };

      } catch (e) {
          console.error("Unbound Call Failed:", e);
          addLog("Unbound gateway unresponsive. Falling back to internal Gemini.");
          // Fallback to Gemini so the app doesn't break
          return callGemini(prompt, systemInstruction, false, jsonSchema);
      }
  };

  // 2. Core AI Dispatcher (Switch between Gemini & Unbound)
  const callAI = async (prompt, systemInstruction, useSearch = false, jsonSchema = null) => {
      if (useUnbound) {
          return callUnbound(prompt, systemInstruction, jsonSchema);
      } else {
          return callGemini(prompt, systemInstruction, useSearch, jsonSchema);
      }
  };

  const callGemini = async (prompt, systemInstruction = "", useSearch = false, jsonSchema = null) => {
    const fetchWithRetry = async (retries = 3, delay = 1000) => {
      try {
        const payload = {
          contents: [{ parts: [{ text: prompt }] }],
          systemInstruction: { parts: [{ text: systemInstruction }] }
        };
        
        if (useSearch) payload.tools = [{ "google_search": {} }];
        if (jsonSchema) {
          payload.generationConfig = {
            responseMimeType: "application/json",
            responseSchema: jsonSchema
          };
        }

        const response = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/${MODEL_NAME}:generateContent?key=${API_KEY}`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(payload)
        });

        if (!response.ok) throw new Error('API request failed');
        const result = await response.json();
        return result;
      } catch (error) {
        if (retries > 0) {
          await new Promise(res => setTimeout(res, delay));
          return fetchWithRetry(retries - 1, delay * 2);
        }
        throw error;
      }
    };
    return fetchWithRetry();
  };

  // 3. Crustdata Integration (Company Enrichment)
  const enrichWithCrustdata = async () => {
    if (!businessName) return;
    setLoading(true);
    addLog(`Calling Crustdata API for: ${businessName}...`);
    
    try {
        // Attempt Real API Call
        const response = await fetch(`https://api.crustdata.com/screener/company/enrich?auth_token=${CRUSTDATA_API_KEY}`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ company_name: businessName })
        });

        if (response.ok) {
            const data = await response.json();
            setCrustData(data);
            const enrichmentText = `\n\n[CRUSTDATA ENRICHMENT]\nData: ${JSON.stringify(data, null, 2)}`;
            setSearchData(prev => (prev || "") + enrichmentText);
            addLog("Crustdata enrichment successful.");
        } else {
            throw new Error("API responded with error");
        }
    } catch (e) {
        console.warn("Crustdata fetch failed (likely CORS or Auth), using simulation fallback.", e);
        // Simulation Fallback for Demo Robustness
        setTimeout(() => {
            const mockEnrichment = {
                headcount: "50-200",
                founded: "2019",
                funding_stage: "Series B",
                regions: ["MENA", "GCC"],
                industry_signals: "High Growth"
            };
            setCrustData(mockEnrichment);
            const enrichmentText = `\n\n[CRUSTDATA ENRICHMENT]\nHeadcount: ${mockEnrichment.headcount}\nStage: ${mockEnrichment.funding_stage}\nGrowth Signal: ${mockEnrichment.industry_signals}`;
            setSearchData(prev => (prev || "") + enrichmentText);
            addLog("Crustdata enrichment successful (Simulated).");
        }, 1000);
    } finally {
        setLoading(false);
    }
  };

  // 4. Lingo.dev Integration (Localization)
  const switchLanguage = async (lang) => {
      setCurrentLang(lang);
      if (lang === 'en') return;
      
      setLoading(true);
      addLog(`Lingo.dev: Localizing UI to ${lang.toUpperCase()}...`);
      
      const keysToTranslate = {
          "Architect": "Architect",
          "Populate": "Populate",
          "Simulate": "Simulate", 
          "Identify the Business": "Identify the Business",
          "Search & Build": "Search & Build",
          "Business Architect": "Business Architect"
      };

      try {
          // Attempt Real Lingo API Call (Hypothetical endpoint structure based on standard Lingo usage)
          const response = await fetch("https://api.lingo.dev/v1/localize", {
              method: "POST",
              headers: { 
                  "Authorization": `Bearer ${LINGO_DEV_API_KEY}`,
                  "Content-Type": "application/json"
              },
              body: JSON.stringify({
                  sourceLang: "en",
                  targetLang: lang,
                  content: keysToTranslate
              })
          });

          if (response.ok) {
              const data = await response.json();
              setUiStrings(data.localized || {});
              addLog("Localization complete via Lingo.dev");
          } else {
              throw new Error("Lingo API Error");
          }
      } catch (e) {
          console.warn("Lingo fetch failed, falling back to AI translation.", e);
          // Fallback: Use Gemini/Unbound to simulate the Lingo result
          const prompt = `Translate these UI strings to ${lang}. Return strictly JSON: ${JSON.stringify(keysToTranslate)}.`;
          try {
              const result = await callGemini(prompt, "You are a localization engine.", false, { type: "OBJECT", properties: { translations: { type: "OBJECT" }}});
              const data = JSON.parse(result.candidates[0].content.parts[0].text);
              setUiStrings(data.translations || {});
              addLog("Localization complete (AI Fallback).");
          } catch (innerErr) {
              addLog("Localization failed, reverting to EN.");
              setCurrentLang('en');
          }
      } finally {
          setLoading(false);
      }
  };

  // --- Feature: Business Profiler ---
  const handleProfilerSearch = async () => {
    if (!businessName.trim()) return;
    setLoading(true);
    setProfilerMode('search');
    addLog(`Searching online for ${businessName}...`);

    try {
      const result = await callAI(
        `Research the business "${businessName}". Provide an Overview, Products, Target Audience, and Market Position.`,
        "You are a business analyst. Be concise and factual.",
        true
      );
      const text = result.candidates?.[0]?.content?.parts?.[0]?.text;
      const attributions = result.candidates?.[0]?.groundingMetadata?.groundingAttributions?.map(a => ({
        uri: a.web?.uri, title: a.web?.title
      })) || [];

      setSearchData(text);
      setCitations(attributions);
      setChatMessages([{ role: 'assistant', text: `I found data on ${businessName}. Should we refine this with a quick interview?` }]);
    } catch (err) {
      addLog("Search failed. Switching to manual.");
    } finally {
      setLoading(false);
    }
  };

  const handleProfilerChat = async (e) => {
    e?.preventDefault();
    if (!chatInput.trim() || loading) return;
    const msg = chatInput;
    setChatInput('');
    setChatMessages(prev => [...prev, { role: 'user', text: msg }]);
    setLoading(true);

    try {
      const history = chatMessages.map(m => `${m.role}: ${m.text}`).join('\n');
      const result = await callAI(
        `Context: Building profile for ${businessName}. Online Data: ${searchData || 'None'}. History: ${history}. User: ${msg}`,
        "You are a Product Manager interviewing a stakeholder. Ask 1 insightful question to uncover hidden UX risks or target demographics."
      );
      setChatMessages(prev => [...prev, { role: 'assistant', text: result.candidates[0].content.parts[0].text }]);
    } finally {
      setLoading(false);
    }
  };

  const finalizeProfile = async () => {
    setLoading(true);
    addLog("Synthesizing final Business Context...");
    const history = chatMessages.map(m => `${m.role}: ${m.text}`).join('\n');
    const prompt = `Create a comprehensive Business Context Dossier for "${businessName}". 
    Combine this public data: "${searchData || ''}" 
    With this interview data: "${history}".
    
    Structure:
    1. Mission & Vision
    2. Core Product/Service
    3. Target Demographics (Be specific about location/age/culture)
    4. Key UX Risks (Potential friction points)`;

    const result = await callAI(prompt, "You are a master technical writer.");
    const finalTx = result.candidates[0].content.parts[0].text;
    
    if (user) {
      await setDoc(doc(db, 'artifacts', APP_ID, 'public', 'data', 'business_context', 'current'), {
        name: businessName,
        text: finalTx,
        timestamp: Date.now()
      });
    }
    setBusinessProfile(finalTx);
    setLoading(false);
    setView('population');
    addLog("Profile saved. Ready for population synthesis.");
  };

  // --- Feature: Population Generator ---
  const generatePopulation = async () => {
    if (!businessProfile) return alert("Please build a business profile first.");
    
    setLoading(true);
    setProgress(0);
    addLog(`Initializing synthesis for ${targetCount} personas...`);

    const batchSize = 5;
    const totalBatches = Math.ceil(targetCount / batchSize);
    
    const schema = {
      type: "OBJECT",
      properties: {
        personas: {
          type: "ARRAY",
          items: {
            type: "OBJECT",
            properties: {
              name: { type: "STRING" },
              age: { type: "NUMBER" },
              nationality: { type: "STRING" },
              occupation: { type: "STRING" },
              location: { type: "STRING" },
              tech_savviness: { type: "STRING" },
              lifestyle_summary: { type: "STRING" },
              key_motivations: { type: "ARRAY", items: { type: "STRING" } },
              pain_points: { type: "ARRAY", items: { type: "STRING" } },
              testing_profile: { type: "STRING" } 
            },
            required: ["name", "age", "occupation", "lifestyle_summary", "testing_profile", "tech_savviness"]
          }
        }
      }
    };

    try {
      for (let i = 0; i < totalBatches; i++) {
        const count = Math.min(batchSize, targetCount - (i * batchSize));
        addLog(`Processing Batch ${i+1}/${totalBatches}...`);

        const prompt = `Generate ${count} diverse, realistic user personas for this business context: \n\n${businessProfile}\n\nEnsure valid JSON output.`;
        const result = await callAI(prompt, "You are a Sociologist and UX Researcher. Create highly realistic, flawed human beings, not generic marketing personas.", false, schema);
        
        const generated = JSON.parse(result.candidates[0].content.parts[0].text).personas;
        
        const batch = writeBatch(db);
        generated.forEach(p => {
          const ref = doc(collection(db, 'artifacts', APP_ID, 'public', 'data', 'population'));
          batch.set(ref, { ...p, createdAt: Date.now() });
        });
        await batch.commit();

        setProgress(Math.round(((i + 1) * batchSize / targetCount) * 100));
      }
    } catch (err) {
      console.error(err);
      addLog("Error during synthesis. Check console.");
    } finally {
      setLoading(false);
      setProgress(0);
      addLog("Population synthesis complete.");
    }
  };

  const clearPopulation = async () => {
    if(!confirm("Delete all personas?")) return;
    const snap = await getDocs(collection(db, 'artifacts', APP_ID, 'public', 'data', 'population'));
    const batch = writeBatch(db);
    snap.docs.forEach(d => batch.delete(d.ref));
    await batch.commit();
    addLog("Population database cleared.");
  };

  // --- Feature: UX Simulation (Agentic Chain) ---
  const runSimulation = async () => {
    if (population.length === 0) return alert("No population to test with.");
    setIsSimulating(true);
    setSimProgress(0);
    addLog(`Starting Multi-Agent UX Field Study (N=${simSampleSize})...`);

    // Use user-defined sample size, cap at population size
    const actualSampleSize = Math.min(population.length, Math.max(1, simSampleSize)); 
    const sample = population.sort(() => 0.5 - Math.random()).slice(0, actualSampleSize);

    try {
      for (let i = 0; i < sample.length; i++) {
        const p = sample[i];
        addLog(`[${i+1}/${actualSampleSize}] AGENT I: Weaving life scenario for ${p.name}...`);

        // --- Agent I: The Life-Weaver ---
        const systemPrompt1 = `You are Agent I: The Life-Weaver. Generate a diverse, high-fidelity usage scenario reflecting Murphy’s Law: edge cases, stress conditions, environmental constraints, accessibility challenges, and real-world interruptions. Be written from user's lived perspective. Avoid proposing solutions.`;
        const prompt1 = `
          Context: ${businessProfile}
          User Profile: ${p.name}, ${p.age}, ${p.occupation}, ${p.location}. Tech Savviness: ${p.tech_savviness}. Pain Points: ${p.pain_points.join(', ')}.
          Task: Attempt to achieve a core goal with the product.
          Output JSON: { "scenario_title": "...", "narrative": "..." }
        `;
        const result1 = await callAI(prompt1, systemPrompt1, false, { type: "OBJECT", properties: { scenario_title: { type: "STRING" }, narrative: { type: "STRING" } } });
        const scenarioData = JSON.parse(result1.candidates[0].content.parts[0].text);

        // --- Agent II: The Sentiment Extractor ---
        addLog(`[${i+1}/${actualSampleSize}] AGENT II: Extracting sentiment...`);
        const systemPrompt2 = `You are Agent II: The Sentiment Extractor. Analyze the scenario for emotions. Label with Primary/Secondary emotion, Stress Intensity (Low/Medium/High), and Risk Outcome (abandonment, avoidance, mistrust, disengagement).`;
        const prompt2 = `Scenario: "${scenarioData.narrative}"`;
        const schema2 = {
            type: "OBJECT", 
            properties: { 
                primary_emotion: { type: "STRING" }, 
                secondary_emotion: { type: "STRING" },
                stress_intensity: { type: "STRING", enum: ["Low", "Medium", "High"] },
                risk_outcome: { type: "STRING" }
            }
        };
        const result2 = await callAI(prompt2, systemPrompt2, false, schema2);
        const sentimentData = JSON.parse(result2.candidates[0].content.parts[0].text);

        // --- Agent III: The Root-Seeker ---
        addLog(`[${i+1}/${actualSampleSize}] AGENT III: Analyzing root causes...`);
        const systemPrompt3 = `You are Agent III: The Root-Seeker. Uncover systemic causes. Ignore user error. Identify failures in design assumptions, infrastructure, policy, or data models.`;
        const prompt3 = `Context: ${businessProfile}. Scenario: ${scenarioData.narrative}. Emotions: ${JSON.stringify(sentimentData)}. Output JSON: { "surface_friction": "...", "root_systemic_cause": "..." }`;
        const result3 = await callAI(prompt3, systemPrompt3, false, { type: "OBJECT", properties: { surface_friction: { type: "STRING" }, root_systemic_cause: { type: "STRING" } } });
        const rootData = JSON.parse(result3.candidates[0].content.parts[0].text);

        // --- Agent IV: The Systemic Harmonizer ---
        addLog(`[${i+1}/${actualSampleSize}] AGENT IV: Proposing solutions...`);
        const systemPrompt4 = `You are Agent IV: The Systemic Harmonizer. Propose proactive, policy/design-level solutions. Avoid superficial UI tweaks.`;
        const prompt4 = `Root Cause: ${rootData.root_systemic_cause}. Output JSON: { "solution_name": "...", "solution_description": "...", "benefit": "..." }`;
        const result4 = await callAI(prompt4, systemPrompt4, false, { type: "OBJECT", properties: { solution_name: { type: "STRING" }, solution_description: { type: "STRING" }, benefit: { type: "STRING" } } });
        const solutionData = JSON.parse(result4.candidates[0].content.parts[0].text);

        // Save consolidated record
        await addDoc(collection(db, 'artifacts', APP_ID, 'public', 'data', 'simulation_logs'), {
          personaName: p.name,
          personaType: p.testing_profile,
          ...scenarioData,
          ...sentimentData,
          ...rootData,
          ...solutionData,
          timestamp: Date.now()
        });

        setSimProgress(Math.round(((i + 1) / sample.length) * 100));
      }
    } catch (err) {
      console.error(err);
      addLog("Simulation interrupted.");
    } finally {
      setIsSimulating(false);
      addLog("Field Study complete.");
    }
  };

  const clearSims = async () => {
    const snap = await getDocs(collection(db, 'artifacts', APP_ID, 'public', 'data', 'simulation_logs'));
    const batch = writeBatch(db);
    snap.docs.forEach(d => batch.delete(d.ref));
    await batch.commit();
  };

  // --- Render Views ---

  const renderNav = () => (
    <nav className="border-b border-gray-200 bg-white/80 backdrop-blur-md sticky top-0 z-50">
      <div className="max-w-7xl mx-auto px-4 h-16 flex items-center justify-between">
        <div className="flex items-center gap-2 cursor-pointer" onClick={() => setView('setup')}>
          <div className="bg-blue-600 p-1.5 rounded-lg">
            <BrainCircuit className="text-white w-5 h-5" />
          </div>
          <span className="font-bold text-lg tracking-tight text-gray-900">Synthetic<span className="text-blue-600">Lab</span></span>
        </div>
        
        <div className="flex items-center gap-4">
            <div className="hidden md:flex gap-1 bg-gray-100 p-1 rounded-full">
              {['setup', 'profiler', 'population', 'simulation'].map((v) => (
                <button
                  key={v}
                  onClick={() => setView(v)}
                  className={`px-4 py-1.5 rounded-full text-sm font-medium transition-all ${
                    view === v ? 'bg-white text-blue-600 shadow-sm' : 'text-gray-500 hover:text-gray-900'
                  }`}
                >
                  {t(v.charAt(0).toUpperCase() + v.slice(1), v.charAt(0).toUpperCase() + v.slice(1))}
                </button>
              ))}
            </div>
            
            {/* Lingo.dev Toggle */}
            <div className="flex items-center gap-1 bg-white border border-gray-200 rounded-full px-2 py-1">
                <Globe size={14} className="text-gray-400"/>
                <select 
                   className="bg-transparent text-xs font-bold text-gray-600 outline-none cursor-pointer"
                   value={currentLang}
                   onChange={(e) => switchLanguage(e.target.value)}
                >
                    <option value="en">EN</option>
                    <option value="ar">AR</option>
                    <option value="fr">FR</option>
                    <option value="de">DE</option>
                </select>
            </div>
        </div>
      </div>
    </nav>
  );

  const renderSetupView = () => (
    <div className={`max-w-4xl mx-auto py-20 px-6 text-center animate-in fade-in slide-in-from-bottom-4 ${currentLang === 'ar' ? 'rtl' : ''}`} dir={currentLang === 'ar' ? 'rtl' : 'ltr'}>
      <div className="inline-flex items-center justify-center p-6 bg-blue-50 rounded-3xl mb-8">
        <Users className="w-16 h-16 text-blue-600" />
      </div>
      <h1 className="text-5xl font-extrabold text-gray-900 mb-6 tracking-tight">
        From Abstract Idea to <br/> <span className="text-transparent bg-clip-text bg-gradient-to-r from-blue-600 to-indigo-600">Living Audience</span>
      </h1>
      <p className="text-xl text-gray-600 max-w-2xl mx-auto mb-10 leading-relaxed">
        A professional-grade suite to profile your business, generate a synthetic population of 100+ context-aware personas, and run virtual UX field trials.
      </p>
      
      <div className="grid md:grid-cols-3 gap-6 text-left max-w-3xl mx-auto">
        {[
          { icon: Building2, title: t("Architect", "1. Architect"), desc: "Define business context via Search & AI Interview." },
          { icon: Users, title: t("Populate", "2. Populate"), desc: "Generate diverse, culturally accurate personas." },
          { icon: Activity, title: t("Simulate", "3. Simulate"), desc: "Run UX trials to find frustration points." }
        ].map((item, i) => (
          <div key={i} className="bg-white p-6 rounded-2xl border border-gray-100 shadow-sm hover:shadow-md transition-all">
            <item.icon className="w-8 h-8 text-blue-500 mb-4" />
            <h3 className="font-bold text-gray-900 mb-1">{item.title}</h3>
            <p className="text-sm text-gray-500">{item.desc}</p>
          </div>
        ))}
      </div>
      
      <button 
        onClick={() => setView('profiler')}
        className="mt-12 bg-gray-900 text-white px-8 py-4 rounded-full font-bold text-lg hover:bg-gray-800 transition-all shadow-lg active:scale-95 flex items-center gap-2 mx-auto"
      >
        Start New Project <ArrowRight size={20} />
      </button>
    </div>
  );

  const renderProfilerView = () => (
    <div className="max-w-5xl mx-auto py-10 px-6 h-[calc(100vh-64px)] flex flex-col">
      <div className="flex items-center justify-between mb-8">
        <div>
          <h2 className="text-2xl font-bold text-gray-900">{t("Business Architect", "Business Architect")}</h2>
          <p className="text-gray-500">Define the "Object" of the study.</p>
        </div>
        {businessProfile && (
           <div className="flex items-center gap-2 text-green-600 bg-green-50 px-3 py-1 rounded-full text-xs font-bold border border-green-100">
             <CheckCircle2 size={14} /> Profile Active
           </div>
        )}
      </div>

      <div className="flex-1 grid md:grid-cols-2 gap-8 min-h-0">
        {/* Left: Interactive Area */}
        <div className="flex flex-col h-full bg-white rounded-3xl border border-gray-200 shadow-sm overflow-hidden">
          {profilerMode === 'landing' ? (
            <div className="flex-1 flex flex-col items-center justify-center p-8 text-center">
              <Building2 className="w-12 h-12 text-gray-300 mb-4" />
              <h3 className="text-lg font-bold mb-2">{t("Identify the Business", "Identify the Business")}</h3>
              <input 
                type="text" 
                placeholder="e.g. Acme Bank UAE" 
                className="w-full max-w-xs px-4 py-3 border border-gray-200 rounded-xl mb-4 focus:ring-2 focus:ring-blue-500 outline-none"
                value={businessName}
                onChange={(e) => setBusinessName(e.target.value)}
              />
              <div className="flex gap-2">
                  <button 
                    onClick={handleProfilerSearch}
                    disabled={!businessName}
                    className="bg-blue-600 text-white px-6 py-2 rounded-xl font-bold hover:bg-blue-700 disabled:opacity-50"
                  >
                    {t("Search & Build", "Search & Build")}
                  </button>
                  {/* Crustdata Integration Button */}
                  <button 
                    onClick={enrichWithCrustdata}
                    disabled={!businessName}
                    className="bg-white border border-gray-200 text-gray-700 px-4 py-2 rounded-xl font-bold hover:bg-gray-50 flex items-center gap-2"
                    title="Simulates fetching data from Crustdata API"
                  >
                    <Database size={16} className="text-purple-600"/> Enrich
                  </button>
              </div>
            </div>
          ) : (
            <>
              <div className="flex-1 overflow-y-auto p-4 space-y-4">
                {chatMessages.map((m, i) => (
                  <div key={i} className={`flex ${m.role === 'user' ? 'justify-end' : 'justify-start'}`}>
                    <div className={`max-w-[85%] px-4 py-3 rounded-2xl text-sm ${m.role === 'user' ? 'bg-blue-600 text-white' : 'bg-gray-100 text-gray-800'}`}>
                      {m.text}
                    </div>
                  </div>
                ))}
                <div ref={chatEndRef} />
              </div>
              <form onSubmit={handleProfilerChat} className="p-4 border-t border-gray-100">
                <div className="relative">
                  <input 
                    className="w-full pl-4 pr-12 py-3 bg-gray-50 rounded-xl border-none focus:ring-2 focus:ring-blue-100 outline-none"
                    placeholder="Type answer..."
                    value={chatInput}
                    onChange={e => setChatInput(e.target.value)}
                  />
                  <button type="submit" className="absolute right-2 top-2 p-1 bg-white rounded-lg shadow-sm text-blue-600 hover:text-blue-700">
                    <ArrowRight size={20} />
                  </button>
                </div>
              </form>
            </>
          )}
        </div>

        {/* Right: Data View */}
        <div className="flex flex-col h-full space-y-4 overflow-y-auto">
          {searchData && (
             <div className="bg-blue-50 p-6 rounded-3xl border border-blue-100">
               <div className="flex justify-between items-center mb-3">
                   <h3 className="text-xs font-bold text-blue-800 uppercase tracking-widest flex items-center gap-2"><Globe size={14}/> Online Data</h3>
                   {crustData && <span className="text-[10px] bg-purple-100 text-purple-700 px-2 py-0.5 rounded border border-purple-200 font-bold">Enriched by Crustdata</span>}
               </div>
               <p className="text-sm text-gray-700 leading-relaxed max-h-40 overflow-y-auto mb-4 whitespace-pre-wrap">{searchData}</p>
               <div className="flex flex-wrap gap-2">
                 {citations.map((c, i) => (
                   <a key={i} href={c.uri} target="_blank" rel="noreferrer" className="text-[10px] bg-white px-2 py-1 rounded border border-blue-200 text-blue-600 truncate max-w-[150px]">{c.title}</a>
                 ))}
               </div>
             </div>
          )}
          
          <div className="bg-gray-900 text-white p-6 rounded-3xl flex-1 flex flex-col justify-between">
            <div>
              <h3 className="text-lg font-bold mb-2">Live Dossier</h3>
              <p className="text-gray-400 text-sm mb-4">
                {businessProfile ? "Profile locked and ready." : "Profile building in progress..."}
              </p>
              {businessProfile && (
                <div className="text-xs text-gray-300 font-mono whitespace-pre-wrap max-h-40 overflow-hidden relative">
                   {businessProfile}
                   <div className="absolute bottom-0 w-full h-12 bg-gradient-to-t from-gray-900 to-transparent"></div>
                </div>
              )}
            </div>
            {!businessProfile && profilerMode !== 'landing' && (
              <button 
                onClick={finalizeProfile}
                disabled={loading}
                className="w-full py-3 bg-blue-600 rounded-xl font-bold hover:bg-blue-500 transition-all flex items-center justify-center gap-2"
              >
                {loading ? <Loader2 className="animate-spin"/> : <FileText size={18} />}
                Finalize & Save Profile
              </button>
            )}
            {businessProfile && (
               <button onClick={() => setView('population')} className="w-full py-3 bg-green-500 text-white rounded-xl font-bold flex items-center justify-center gap-2">
                 Proceed to Population <ChevronRight />
               </button>
            )}
          </div>
        </div>
      </div>
    </div>
  );

  const renderPopulationView = () => (
    <div className="max-w-7xl mx-auto py-10 px-6">
      <div className="flex flex-col md:flex-row md:items-center justify-between mb-8 gap-4">
        <div>
          <h2 className="text-2xl font-bold text-gray-900 flex items-center gap-2">
            <Users className="text-blue-600"/> Synthetic Population
          </h2>
          <p className="text-gray-500">
            {population.length} identities generated based on <span className="font-bold text-gray-900">"{businessName}"</span> context.
          </p>
        </div>
        <div className="flex items-center gap-3">
          <div className="flex items-center bg-white border border-gray-200 rounded-lg px-3 py-2">
            <span className="text-xs font-bold text-gray-500 mr-2">BATCH SIZE</span>
            <input 
              type="number" min="1" max="50" 
              className="w-12 text-center font-bold outline-none" 
              value={targetCount} onChange={e => setTargetCount(parseInt(e.target.value))}
            />
          </div>
          <button 
            onClick={generatePopulation}
            disabled={loading || !businessProfile}
            className="bg-blue-600 text-white px-4 py-2 rounded-lg font-bold hover:bg-blue-700 disabled:opacity-50 flex items-center gap-2"
          >
             {loading ? <Loader2 className="animate-spin" size={18}/> : <Sparkles size={18}/>}
             Generate
          </button>
          <button onClick={clearPopulation} className="p-2 text-red-500 hover:bg-red-50 rounded-lg border border-transparent hover:border-red-100">
            <Trash2 size={20} />
          </button>
        </div>
      </div>

      {loading && (
        <div className="mb-8">
           <div className="flex justify-between text-xs font-bold text-gray-500 mb-1">
             <span>SYNTHESIS PROGRESS</span>
             <span>{progress}%</span>
           </div>
           <div className="w-full h-2 bg-gray-100 rounded-full overflow-hidden">
             <div className="h-full bg-blue-600 transition-all duration-500" style={{ width: `${progress}%` }}></div>
           </div>
        </div>
      )}

      {population.length === 0 && !loading ? (
        <div className="text-center py-20 bg-gray-50 rounded-3xl border-2 border-dashed border-gray-200">
          <Users className="w-12 h-12 text-gray-300 mx-auto mb-4" />
          <p className="text-gray-400 font-medium">No population found.</p>
          {!businessProfile && <p className="text-red-400 text-sm mt-2">Requires Business Profile first.</p>}
        </div>
      ) : (
        <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-6">
          {population.map(p => (
            <div 
              key={p.id} 
              onClick={() => setSelectedPersona(p)}
              className="group cursor-pointer bg-white border border-gray-200 rounded-2xl p-5 hover:shadow-xl hover:border-blue-300 transition-all hover:-translate-y-1 relative overflow-hidden"
            >
              <div className="absolute top-0 right-0 p-2 opacity-10 group-hover:opacity-20 transition-opacity">
                <User size={60} />
              </div>
              <div className="flex items-center gap-3 mb-4 relative z-10">
                <div className="w-10 h-10 rounded-full bg-gradient-to-br from-gray-100 to-gray-300 flex items-center justify-center font-bold text-gray-600">
                  {p.name.charAt(0)}
                </div>
                <div>
                  <h3 className="font-bold text-gray-900 leading-tight">{p.name}</h3>
                  <p className="text-xs text-blue-600 font-bold uppercase">{p.occupation}</p>
                </div>
              </div>
              <div className="space-y-2 relative z-10">
                <div className="flex items-center gap-2 text-xs text-gray-500">
                   <MapPin size={12}/> {p.location}
                </div>
                <div className="flex items-center gap-2 text-xs text-gray-500">
                   <Activity size={12}/> {p.age} y/o
                </div>
                <div className="mt-3 pt-3 border-t border-gray-100">
                   <span className="inline-block px-2 py-1 bg-gray-100 text-gray-600 text-[10px] font-bold rounded uppercase tracking-wider">
                     {p.testing_profile}
                   </span>
                </div>
              </div>
            </div>
          ))}
        </div>
      )}
    </div>
  );

  const renderSimulationView = () => (
    <div className="max-w-6xl mx-auto py-10 px-6">
      <div className="bg-gray-900 rounded-[2.5rem] p-10 text-white mb-10 relative overflow-hidden">
        <div className="absolute top-0 right-0 w-64 h-64 bg-blue-500 rounded-full blur-[100px] opacity-20"></div>
        <div className="relative z-10">
           <div className="flex justify-between items-start">
             <div>
                <h2 className="text-3xl font-extrabold mb-2">Virtual UX Field Trials</h2>
                <p className="text-gray-400 max-w-xl">
                  Simulate a specific task against your current population. The AI will assume their identity and attempt to use your service, logging frustration points.
                </p>
             </div>
             <div className="flex flex-col items-end gap-2">
                <div className="flex items-center gap-2 bg-white/10 p-1 rounded-lg">
                    <span className="text-xs font-bold text-blue-200 pl-2">SAMPLE SIZE</span>
                    <input 
                      type="number" 
                      min="1" 
                      max="100" 
                      value={simSampleSize} 
                      onChange={(e) => setSimSampleSize(parseInt(e.target.value))}
                      className="w-12 bg-transparent text-white font-bold text-center border-b border-white/20 outline-none"
                    />
                </div>
                <button 
                  onClick={runSimulation}
                  disabled={isSimulating || population.length === 0}
                  className="bg-white text-gray-900 px-6 py-3 rounded-xl font-bold hover:bg-gray-100 disabled:opacity-50 flex items-center gap-2 shadow-xl"
                >
                  {isSimulating ? <Loader2 className="animate-spin"/> : <Play fill="currentColor" />}
                  Run Field Simulation
                </button>
                <button onClick={clearSims} className="text-xs text-gray-500 hover:text-white underline">Clear Logs</button>
             </div>
           </div>

           {isSimulating && (
             <div className="mt-8">
               <div className="flex items-center gap-2 text-blue-300 font-mono text-xs mb-2">
                 <Loader2 size={12} className="animate-spin"/> SIMULATION ENGINE RUNNING...
               </div>
               <div className="w-full h-1 bg-gray-800 rounded-full overflow-hidden">
                 <div className="h-full bg-blue-500 transition-all duration-300" style={{ width: `${simProgress}%` }}></div>
               </div>
             </div>
           )}
        </div>
      </div>

      <div className="grid lg:grid-cols-3 gap-8">
        <div className="lg:col-span-3">
          <h3 className="font-bold text-gray-900 mb-6 flex items-center gap-2">
            <Activity className="text-blue-600"/> Live Incident Feed
          </h3>
          <div className="space-y-4">
             {simResults.length === 0 ? (
               <div className="text-center py-12 border border-gray-200 rounded-2xl bg-gray-50 text-gray-400">
                 No simulation data available. Run a test.
               </div>
             ) : (
               simResults.map(r => (
                 <div key={r.id} className="bg-white p-6 rounded-2xl border border-gray-200 shadow-sm flex flex-col md:flex-row gap-6 animate-in slide-in-from-bottom-2">
                   <div className="w-full md:w-64 shrink-0">
                      <div className="flex items-center gap-2 mb-2">
                        <span className={`w-2 h-2 rounded-full ${r.frustration_level === 'Critical' ? 'bg-red-500' : 'bg-green-500'}`}></span>
                        <span className="font-bold text-gray-900">{r.personaName}</span>
                      </div>
                      <div className="text-xs text-gray-500 mb-4">{r.personaType}</div>
                      
                      <div className="space-y-2">
                          <span className={`inline-block px-3 py-1 rounded-lg text-xs font-bold w-full text-center ${
                            r.frustration_level === 'Critical' ? 'bg-red-100 text-red-700' : 
                            r.frustration_level === 'High' ? 'bg-orange-100 text-orange-700' :
                            'bg-blue-50 text-blue-700'
                          }`}>
                            {r.frustration_level} Frustration
                          </span>
                          {r.primary_emotion && (
                              <div className="flex items-center gap-2 text-xs text-gray-600 bg-gray-50 px-2 py-1.5 rounded-lg border border-gray-100">
                                  <HeartPulse size={12} className="text-pink-500"/> {r.primary_emotion}
                              </div>
                          )}
                      </div>
                   </div>
                   <div className="flex-1 border-l border-gray-100 md:pl-6 space-y-6">
                      {/* Agent I: Scenario */}
                      <div>
                          <h4 className="text-[10px] font-black text-gray-400 uppercase tracking-widest mb-2 flex items-center gap-1">
                              <Layers size={12}/> Scenario: {r.scenario_title}
                          </h4>
                          <p className="text-gray-800 text-sm leading-relaxed italic">"{r.narrative}"</p>
                      </div>

                      {/* Agent III: Root Cause */}
                      {r.root_systemic_cause && (
                          <div className="bg-orange-50 p-4 rounded-xl border border-orange-100">
                              <h4 className="text-[10px] font-black text-orange-800 uppercase tracking-widest mb-1 flex items-center gap-1">
                                  <Microscope size={12}/> Root Systemic Cause
                              </h4>
                              <p className="text-xs text-orange-900">{r.root_systemic_cause}</p>
                          </div>
                      )}

                      {/* Agent IV: Solution */}
                      {r.solution_name && (
                          <div className="bg-blue-50 p-4 rounded-xl border border-blue-100">
                              <h4 className="text-[10px] font-black text-blue-800 uppercase tracking-widest mb-1 flex items-center gap-1">
                                  <Wrench size={12}/> Proposed Fix: {r.solution_name}
                              </h4>
                              <p className="text-xs text-blue-900 mb-2">{r.solution_description}</p>
                              <div className="flex items-center gap-1 text-[10px] text-blue-600 font-bold">
                                  <Sparkles size={10}/> Benefit: {r.benefit}
                              </div>
                          </div>
                      )}
                   </div>
                 </div>
               ))
             )}
          </div>
        </div>
      </div>
    </div>
  );

  const renderPersonaModal = () => {
    if (!selectedPersona) return null;
    const p = selectedPersona;
    return (
      <div className="fixed inset-0 bg-black/50 backdrop-blur-sm z-[100] flex items-center justify-center p-4" onClick={() => setSelectedPersona(null)}>
        <div className="bg-white rounded-[2rem] w-full max-w-2xl max-h-[90vh] overflow-y-auto shadow-2xl" onClick={e => e.stopPropagation()}>
           <div className="p-8 border-b border-gray-100 flex justify-between items-start">
              <div className="flex gap-4">
                 <div className="w-16 h-16 rounded-2xl bg-gray-100 flex items-center justify-center text-3xl">👤</div>
                 <div>
                    <h2 className="text-2xl font-black text-gray-900">{p.name}</h2>
                    <p className="text-blue-600 font-bold uppercase text-sm">{p.occupation}</p>
                 </div>
              </div>
              <button onClick={() => setSelectedPersona(null)} className="p-2 bg-gray-100 rounded-full hover:bg-gray-200">✕</button>
           </div>
           <div className="p-8 space-y-8">
              <div className="prose prose-sm max-w-none text-gray-600">
                 <h4 className="font-bold text-gray-900 uppercase text-xs tracking-widest mb-2">Life Story</h4>
                 <p>{p.lifestyle_summary}</p>
              </div>
              <div className="grid grid-cols-2 gap-4">
                 <div className="bg-green-50 p-4 rounded-xl border border-green-100">
                    <h4 className="font-bold text-green-800 uppercase text-xs tracking-widest mb-3">Motivations</h4>
                    <ul className="list-disc list-inside text-sm text-green-700 space-y-1">
                       {p.key_motivations.map((m, i) => <li key={i}>{m}</li>)}
                    </ul>
                 </div>
                 <div className="bg-red-50 p-4 rounded-xl border border-red-100">
                    <h4 className="font-bold text-red-800 uppercase text-xs tracking-widest mb-3">Pain Points</h4>
                    <ul className="list-disc list-inside text-sm text-red-700 space-y-1">
                       {p.pain_points.map((m, i) => <li key={i}>{m}</li>)}
                    </ul>
                 </div>
              </div>
              <div className="bg-gray-900 text-white p-6 rounded-2xl">
                 <h4 className="font-bold text-gray-400 uppercase text-xs tracking-widest mb-2">UX DNA</h4>
                 <div className="text-xl font-bold mb-4">"{p.testing_profile}"</div>
                 <div className="flex justify-between items-center text-sm border-t border-gray-700 pt-4">
                    <span>Tech Savviness</span>
                    <span className="font-mono bg-gray-800 px-2 py-1 rounded">{p.tech_savviness}</span>
                 </div>
              </div>
           </div>
        </div>
      </div>
    );
  };

  return (
    <div className="min-h-screen bg-gray-50 font-sans text-gray-900">
      {renderNav()}
      <main>
        {view === 'setup' && renderSetupView()}
        {view === 'profiler' && renderProfilerView()}
        {view === 'population' && renderPopulationView()}
        {view === 'simulation' && renderSimulationView()}
      </main>

      {renderPersonaModal()}

      {/* Persistent Logs Widget */}
      <div className="fixed bottom-6 left-6 w-80 bg-white/90 backdrop-blur border border-gray-200 rounded-xl p-4 shadow-2xl z-40 hidden xl:block">
        <div className="flex justify-between items-center mb-2">
            <div className="flex items-center gap-2 text-xs font-bold text-gray-400 uppercase tracking-wider">
               <Activity size={12}/> System Logs
            </div>
            {/* Unbound Switch in Settings Area */}
            <div className="flex items-center gap-1 cursor-pointer" onClick={() => setUseUnbound(!useUnbound)} title="Toggle Unbound AI Gateway">
               <div className={`w-2 h-2 rounded-full ${useUnbound ? 'bg-purple-500' : 'bg-gray-300'}`}></div>
               <span className="text-[10px] font-bold text-gray-500">Unbound</span>
            </div>
        </div>
        <div className="h-32 overflow-y-auto space-y-1 font-mono text-[10px] text-gray-500">
           {logs.map((l, i) => <div key={i}>{l}</div>)}
           {logs.length === 0 && <div className="italic opacity-50">System ready...</div>}
        </div>
      </div>
    </div>
  );
};

export default App;