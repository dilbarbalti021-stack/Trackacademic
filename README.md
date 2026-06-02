 import React, { useState, useEffect, useMemo, useRef, useCallback } from 'react';
import { 
  BookOpen, 
  CheckCircle, 
  ChevronDown, 
  ChevronRight, 
  Plus, 
  BarChart3, 
  Calendar, 
  AlertCircle, 
  MessageSquare, 
  FileText, 
  Upload, 
  Target, 
  Clock, 
  Zap, 
  Trash2, 
  ExternalLink, 
  BrainCircuit, 
  Lightbulb, 
  Volume2, 
  X, 
  Play, 
  Search, 
  Layers, 
  ShieldCheck, 
  Sigma,
  TrendingUp,
  FileQuestion,
  Filter,
  Flame,
  Download,
  Maximize2,
  Minimize2,
  Sparkles
} from 'lucide-react';
import {
  Chart as ChartJS,
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  BarElement,
  Title,
  Tooltip,
  Legend,
  ArcElement
} from 'chart.js';
import { Bar, Doughnut } from 'react-chartjs-2';

ChartJS.register(CategoryScale, LinearScale, PointElement, LineElement, BarElement, ArcElement, Title, Tooltip, Legend);

// Global Memory Cache for large PDF binaries to avoid localStorage limits
const pdfBinaryCache = new Map();

// --- External Dependencies ---
const loadScript = (src) => new Promise((resolve) => {
  if (document.querySelector(`script[src="${src}"]`)) return resolve();
  const script = document.createElement('script');
  script.src = src;
  script.onload = resolve;
  document.head.appendChild(script);
});

const loadStyle = (href) => {
  if (document.querySelector(`link[href="${href}"]`)) return;
  const link = document.createElement('link');
  link.rel = 'stylesheet';
  link.href = href;
  document.head.appendChild(link);
};

const apiKey = ""; 
const MODEL_NAME = "gemini-2.5-flash-preview-09-2025";
const TTS_MODEL = "gemini-2.5-flash-preview-tts";

const PROB_COLORS = {
  High: 'bg-rose-100 text-rose-700 border-rose-200',
  Medium: 'bg-amber-100 text-amber-700 border-amber-200',
  Low: 'bg-emerald-100 text-emerald-700 border-emerald-200'
};

// --- Render Mathematical Formulas (LaTeX support) ---
const FormulaRenderer = ({ text }) => {
  if (!text || typeof text !== 'string') return null;
  const parts = text.split(/(\$\$.*?\Split|\$.*?\$)/g);
  return (
    <div className="whitespace-pre-wrap leading-relaxed">
      {parts.map((part, i) => {
        if (!part) return null;
        if (part.startsWith('$') && window.katex) {
          const isDisplay = part.startsWith('$$');
          const formula = isDisplay ? part.slice(2, -2) : part.slice(1, -1);
          try {
            const html = window.katex.renderToString(formula, { displayMode: isDisplay, throwOnError: false });
            return <span key={i} dangerouslySetInnerHTML={{ __html: html }} />;
          } catch (e) { return <span key={i}>{part}</span>; }
        }
        return <span key={i}>{part}</span>;
      })}
    </div>
  );
};

const App = () => {
  // --- State ---
  const [subjects, setSubjects] = useState(() => {
    const saved = localStorage.getItem('trackademic_v5_meta');
    return saved ? JSON.parse(saved) : [];
  });
  const [activeSubjectId, setActiveSubjectId] = useState(null);
  const [isAnalyzing, setIsAnalyzing] = useState(false);
  const [chatOpen, setChatOpen] = useState(false);
  const [messages, setMessages] = useState([]);
  const [inputMessage, setInputMessage] = useState("");
  const [view, setView] = useState('dashboard'); 
  const [activeTab, setActiveTab] = useState('syllabus'); // syllabus | pastpapers | predictions
  const [pdfWorkerReady, setPdfWorkerReady] = useState(false);
  const [filterProb, setFilterProb] = useState('All');
  
  // Custom Toasts State
  const [toasts, setToasts] = useState([]);

  // Modals & Enhanced Features State
  const [aiModal, setAiModal] = useState({ isOpen: false, title: "", content: null, isLoading: false, sourcePage: null });
  const [pdfViewer, setPdfViewer] = useState({ isOpen: false, file: null, page: 1, zoom: 1.0, searchTerm: "" });
  const [guessPaper, setGuessPaper] = useState({ isOpen: false, content: null, isLoading: false });
  const [scheduleModal, setScheduleModal] = useState({ isOpen: false, plan: null, isLoading: false });
  const [isSpeaking, setIsSpeaking] = useState(false);
  const [activeSpeakerSection, setActiveSpeakerSection] = useState(null);

  // --- Helpers & Business Logic (Declared up top to prevent hoisting / TDZ issues) ---
  const calculateProgress = useCallback((subject) => {
    if (!subject?.chapters?.length) return 0;
    let total = 0, done = 0;
    subject.chapters.forEach(ch => {
      ch.topics.forEach(t => { total++; if (t.done) done++; });
    });
    return total === 0 ? 0 : Math.round((done / total) * 100);
  }, []);

  const getOverallProgress = useCallback(() => {
    if (subjects.length === 0) return 0;
    const sum = subjects.reduce((acc, s) => acc + calculateProgress(s), 0);
    return Math.round(sum / subjects.length);
  }, [subjects, calculateProgress]);

  const updateDeadline = useCallback((id, newDate) => {
    setSubjects(prev => prev.map(s => s.id === id ? { ...s, deadline: newDate } : s));
  }, []);

  const getTopicScore = useCallback((topic) => {
    let score = topic.examProbability === 'High' ? 40 : 20;
    score += (topic.pastPaperFrequency || 0) * 15;
    return Math.min(100, score);
  }, []);

  const getClassification = useCallback((score) => {
    if (score >= 60) return { label: 'Most Expected', color: 'text-rose-600 bg-rose-50 border-rose-100' };
    if (score >= 30) return { label: 'Likely', color: 'text-amber-600 bg-amber-50 border-amber-100' };
    return { label: 'Rare', color: 'text-slate-500 bg-slate-50 border-slate-100' };
  }, []);

  const toggleTopic = useCallback((sid, cid, tid) => {
    setSubjects(prev => prev.map(s => s.id === sid ? { 
      ...s, 
      chapters: s.chapters.map(ch => ch.id === cid ? { 
        ...ch, 
        topics: ch.topics.map(t => t.id === tid ? { ...t, done: !t.done } : t) 
      } : ch) 
    } : s));
  }, []);

  const deleteSubject = useCallback((id) => {
    setSubjects(prev => prev.filter(s => s.id !== id));
    if (activeSubjectId === id) { 
      setView('dashboard'); 
      setActiveSubjectId(null); 
    }
  }, [activeSubjectId]);

  // --- Initial Setup ---
  useEffect(() => {
    const init = async () => {
      try {
        await loadScript('https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.min.js');
        window.pdfjsLib.GlobalWorkerOptions.workerSrc = 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.worker.min.js';
        
        loadStyle('https://cdn.jsdelivr.net/npm/katex@0.16.9/dist/katex.min.css');
        await loadScript('https://cdn.jsdelivr.net/npm/katex@0.16.9/dist/katex.min.js');
        
        // Dynamically load jsPDF and html2canvas for PDF download
        await loadScript('https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js');
        await loadScript('https://cdnjs.cloudflare.com/ajax/libs/html2canvas/1.4.1/html2canvas.min.js');
        
        setPdfWorkerReady(true);
      } catch (err) {
        addToast("Failed to load third-party dependencies.", "error");
      }
    };
    init();
  }, []);

  useEffect(() => {
    try {
      localStorage.setItem('trackademic_v5_meta', JSON.stringify(subjects));
    } catch (e) {
      addToast("Failed to persist data. LocalStorage quota exceeded.", "error");
    }
  }, [subjects]);

  const activeSubject = useMemo(() => subjects.find(s => s.id === activeSubjectId), [subjects, activeSubjectId]);

  // --- Toast Manager ---
  const addToast = useCallback((message, type = "success") => {
    const id = Date.now();
    setToasts(prev => [...prev, { id, message, type }]);
    setTimeout(() => {
      setToasts(prev => prev.filter(t => t.id !== id));
    }, 4000);
  }, []);

  // --- Weighted Probability Scoring Engine ---
  const calculateProbability = useCallback((topicName, pastPaperFreq = 0) => {
    let score = 20; // Base score

    // Factor A: Academic Keywords Heuristic
    const academicKeywords = {
      "law": 30,
      "theorem": 30,
      "principle": 25,
      "derivation": 25,
      "proof": 20,
      "formula": 15,
      "equation": 15,
      "definition": 10,
      "cycle": 10
    };

    const lowerName = topicName.toLowerCase();
    Object.entries(academicKeywords).forEach(([key, weight]) => {
      if (lowerName.includes(key)) {
        score += weight;
      }
    });

    // Factor B: Complex topic name length
    if (topicName.length > 25) {
      score += 10;
    }

    // Factor C: Past Paper Frequency Weight
    if (pastPaperFreq === 1) score += 20;
    else if (pastPaperFreq === 2) score += 35;
    else if (pastPaperFreq >= 3) score += 50;

    // Cap the score at 100
    score = Math.min(100, Math.max(10, score));

    // Classification boundaries
    let level = 'Low';
    if (score >= 75) level = 'High';
    else if (score >= 40) level = 'Medium';

    return {
      level,
      score,
      reason: `Assigned index of ${score}% based on academic term weight and ${pastPaperFreq} matched exam instance(s).`
    };
  }, []);

  // --- PDF & Past Paper Processing ---
  const extractPdfData = async (file) => {
    const arrayBuffer = await file.arrayBuffer();
    const pdf = await window.pdfjsLib.getDocument({ data: arrayBuffer }).promise;
    let text = "";
    const maxProcess = Math.min(pdf.numPages, 50);
    for (let i = 1; i <= maxProcess; i++) {
      const page = await pdf.getPage(i);
      const content = await page.getTextContent();
      text += ` [P${i}] ` + content.items.map(it => it.str).join(' ');
    }
    return text;
  };

  const handleSyllabusUpload = async (e) => {
    const file = e.target.files[0];
    if (!file || !pdfWorkerReady) return;
    setIsAnalyzing(true);
    try {
      const text = await extractPdfData(file);
      const result = await callAI(
        `Extract Syllabus from textbook content: ${text.substring(0, 15000)}`,
        `You are a Syllabus Architect. Respond in strictly valid JSON format. Outline clean logical chapters and distinct topics.
        JSON format: { "subject": string, "chapters": [{ "name": string, "page": number, "topics": [{ "name": string, "page": number }] }] }`
      );

      if (result) {
        const reader = new FileReader();
        reader.readAsDataURL(file);
        reader.onload = () => {
          const sid = Date.now().toString();
          pdfBinaryCache.set(sid, reader.result);
          const newSub = {
            id: sid,
            name: result.subject || file.name.replace('.pdf', ''),
            deadline: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000).toISOString().split('T')[0],
            pastPapers: [],
            chapters: result.chapters.map(ch => ({
              ...ch,
              id: Math.random().toString(36).substr(2, 9),
              isExpanded: true,
              topics: ch.topics.map(t => {
                const prob = calculateProbability(t.name, 0);
                return { 
                  ...t, 
                  id: Math.random().toString(36).substr(2, 9), 
                  done: false, 
                  examProbability: prob.level, 
                  probabilityScore: prob.score,
                  probReason: prob.reason,
                  pastPaperFrequency: 0,
                  pastQuestions: []
                };
              })
            }))
          };
          setSubjects([...subjects, newSub]);
          setActiveSubjectId(sid);
          setView('subject');
          addToast("Syllabus PDF integrated with weighted probability analysis.", "success");
          setIsAnalyzing(false);
        };
      } else {
        addToast("Could not interpret syllabus structure. Please try a different document.", "error");
        setIsAnalyzing(false);
      }
    } catch (e) { 
      addToast("Failed to parse PDF document.", "error");
      setIsAnalyzing(false); 
    }
  };

  // --- Past Paper Intelligence Synonyms Heuristics ---
  const handlePastPaperUpload = async (e) => {
    const file = e.target.files[0];
    if (!file || !activeSubject) return;
    setIsAnalyzing(true);
    try {
      const text = await extractPdfData(file);
      const syllabusTopics = activeSubject.chapters.flatMap(c => c.topics.map(t => t.name));
      const prompt = `Syllabus topics list: ${syllabusTopics.join(', ')}. 
      Find any occurrences, partial matches, synonymous descriptions, or laws tested in this past paper text: ${text.substring(0, 15000)}.
      Return JSON format: { "matches": [{ "topicName": string, "questionSnippet": string, "year": string, "marks": number }] }`;
      
      const analysis = await callAI(prompt, "You are an expert academic examiner matching past papers with curriculum. Use intelligent fuzzy semantics to resolve synonyms.");
      
      if (analysis && analysis.matches) {
        setSubjects(prev => prev.map(s => {
          if (s.id !== activeSubject.id) return s;
          const updatedChapters = s.chapters.map(ch => ({
            ...ch,
            topics: ch.topics.map(t => {
              const matches = analysis.matches.filter(m => m.topicName.toLowerCase() === t.name.toLowerCase() || t.name.toLowerCase().includes(m.topicName.toLowerCase()));
              if (matches.length > 0) {
                const freq = t.pastPaperFrequency + matches.length;
                const newProb = calculateProbability(t.name, freq);
                return {
                  ...t,
                  pastPaperFrequency: freq,
                  examProbability: newProb.level,
                  probabilityScore: newProb.score,
                  probReason: newProb.reason,
                  pastQuestions: [...(t.pastQuestions || []), ...matches.map(m => ({ snippet: m.questionSnippet, year: m.year || '2026', marks: m.marks || 5 }))]
                };
              }
              return t;
            })
          }));
          return {
            ...s,
            pastPapers: [...s.pastPapers, { id: Date.now(), name: file.name }],
            chapters: updatedChapters
          };
        }));
        addToast("Past paper parsed. Cross-reference frequency maps updated successfully.", "success");
      } else {
        addToast("Analyzing past paper returned an invalid schema. Defaulting to general matching.", "error");
      }
      setIsAnalyzing(false);
    } catch (e) { 
      addToast("Error parsing the Past Paper binary structure.", "error");
      setIsAnalyzing(false); 
    }
  };

  // --- AI Model Helper ---
  const callAI = async (prompt, systemInstruction = "", asJson = true) => {
    try {
      const response = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/${MODEL_NAME}:generateContent?key=${apiKey}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          contents: [{ parts: [{ text: prompt }] }],
          systemInstruction: { parts: [{ text: systemInstruction }] },
          generationConfig: asJson ? { responseMimeType: "application/json" } : {}
        })
      });
      const data = await response.json();
      const text = data.candidates?.[0]?.content?.parts?.[0]?.text || "";
      return asJson ? JSON.parse(text) : text;
    } catch (error) { 
      addToast("Network connection with TracKademic AI interrupted.", "error");
      return null; 
    }
  };

  // --- Optimize Study Schedule Engine ---
  const optimizeStudySchedule = useCallback(async () => {
    if (!activeSubject) return;
    setScheduleModal({ isOpen: true, plan: null, isLoading: true });

    const remainingTopics = activeSubject.chapters.flatMap(c => c.topics).filter(t => !t.done);
    const daysLeft = Math.max(1, Math.ceil((new Date(activeSubject.deadline) - new Date()) / (1000 * 60 * 60 * 24)));

    // Sort remaining topics based on priority score (highest first)
    const sortedTopics = [...remainingTopics].sort((a, b) => getTopicScore(b) - getTopicScore(a));

    const prompt = `Given these topics left to study for ${activeSubject.name} (${daysLeft} days remaining):
    ${sortedTopics.map(t => `- Name: "${t.name}", Probability Score: ${t.probabilityScore}%, Past Paper Matches: ${t.pastPaperFrequency}`).join('\n')}
    Distribute these logically over ${Math.min(daysLeft, 7)} days of an intense daily study schedule. 
    Calculate expected hours per day based on complexity. Provide actionable exam-prep focus advice.
    Return strictly in JSON format:
    {
      "estimatedCompletionDate": string,
      "recommendedHoursPerDay": number,
      "studyPlan": [
        { "day": number, "topics": [string], "focusMetric": string, "hoursRequired": number }
      ],
      "aiSummaryAdvice": string
    }`;

    try {
      const result = await callAI(prompt, "You are a senior academic strategist who schedules tasks based on probability index and structural pacing.");
      if (result) {
        setScheduleModal({ isOpen: true, plan: result, isLoading: false });
        addToast("Curriculum pacing schedule generated successfully.", "success");
      } else {
        setScheduleModal(prev => ({ ...prev, isLoading: false, isOpen: false }));
        addToast("Error parsing schedule generation.", "error");
      }
    } catch (e) {
      setScheduleModal(prev => ({ ...prev, isLoading: false, isOpen: false }));
    }
  }, [activeSubject, getTopicScore, addToast]);

  // --- Real downloadable PDF Guess Paper Generator ---
  const downloadGuessPaperPdf = async () => {
    if (!guessPaper.content || !activeSubject) return;
    addToast("Generating your downloadable PDF document...", "info");
    
    try {
      const element = document.getElementById('guess-paper-printable');
      if (!element) return;

      const canvas = await window.html2canvas(element, {
        scale: 2,
        useCORS: true,
        logging: false
      });

      const imgData = canvas.toDataURL('image/png');
      const { jsPDF } = window.jspdf;
      const pdf = new jsPDF('p', 'mm', 'a4');
      
      const imgWidth = 210; // A4 size
      const pageHeight = 295;
      const imgHeight = (canvas.height * imgWidth) / canvas.width;
      let heightLeft = imgHeight;
      let position = 0;

      pdf.addImage(imgData, 'PNG', 0, position, imgWidth, imgHeight);
      heightLeft -= pageHeight;

      while (heightLeft >= 0) {
        position = heightLeft - imgHeight;
        pdf.addPage();
        pdf.addImage(imgData, 'PNG', 0, position, imgWidth, imgHeight);
        heightLeft -= pageHeight;
      }

      pdf.save(`${activeSubject.name.replace(/\s+/g, '_')}_TracKademic_Guess_Paper.pdf`);
      addToast("Guess paper successfully downloaded.", "success");
    } catch (err) {
      addToast("Failed to compile guess paper as a PDF file.", "error");
    }
  };

  const generateGuessPaper = async () => {
    setGuessPaper({ isOpen: true, content: null, isLoading: true });
    const topics = activeSubject.chapters.flatMap(c => c.topics)
      .sort((a, b) => getTopicScore(b) - getTopicScore(a))
      .slice(0, 15);
    
    const prompt = `Using these highly expected syllabus chapters and past paper concepts: ${topics.map(t => `${t.name} (Frequency: ${t.pastPaperFrequency}, Score: ${t.probabilityScore}%)`).join(', ')}, generate a rigorous and realistic predicted exam guess paper for the course "${activeSubject.name}".
    Output strictly in JSON: { "mcqs": [string], "shortQuestions": [string], "longQuestions": [string] }`;

    const result = await callAI(prompt, "You are a Chief Academic Assessor.");
    setGuessPaper(prev => ({ ...prev, content: result, isLoading: false }));
  };

  // --- Presentation-Style Explain Topic Modal ---
  const explainTopicDetailed = async (topic) => {
    setAiModal({ isOpen: true, title: topic.name, content: null, isLoading: true, sourcePage: topic.page });
    
    const prompt = `Perform a rigorous grounded analysis on "${topic.name}" from ${activeSubject.name}. Break the response down into presentation slides/notes format. Include latex formulas where useful (wrap with $ or $$).
    Return STRICTLY as a JSON object with this precise structure:
    {
      "title": "Topic Title",
      "summary": "An elegantly written high-level conceptual summary",
      "keyPoints": ["Key point 1 explaining underlying principles", "Key point 2 showing implications"],
      "importantDefinitions": [{"term": "Term Name", "definition": "Direct precise definitions"}],
      "formulas": [{"name": "Formula Name", "latex": "E = mc^2"}],
      "examTips": ["Crucial exam tip 1", "Common structural mistakes students make"],
      "commonQuestions": [{"question": "Potential exam question related to this?", "marks": 5, "answer": "Suggested structured key marking criteria answer."}],
      "memoryTricks": [{"concept": "Difficult part to recall", "trick": "Mnemonic or physical explanation"}]
    }`;

    try {
      const result = await callAI(prompt, "You are an expert curriculum systems educator.");
      if (result) {
        setAiModal(prev => ({ ...prev, content: result, isLoading: false }));
      } else {
        setAiModal(prev => ({ ...prev, isOpen: false }));
        addToast("Error parsing explanation response.", "error");
      }
    } catch (e) {
      setAiModal(prev => ({ ...prev, isOpen: false }));
    }
  };

  const speakTextGrounded = async (text, sectionId) => {
    if (isSpeaking) {
      setIsSpeaking(false);
      setActiveSpeakerSection(null);
      return;
    }
    setIsSpeaking(true);
    setActiveSpeakerSection(sectionId);
    try {
      const response = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/${TTS_MODEL}:generateContent?key=${apiKey}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          contents: [{ parts: [{ text: `Summarize clearly: ${text.substring(0, 450)}` }] }],
          generationConfig: { 
            responseModalities: ["AUDIO"],
            speechConfig: { voiceConfig: { prebuiltVoiceConfig: { voiceName: "Puck" } } }
          }
        })
      });
      const result = await response.json();
      const audioData = result.candidates[0].content.parts[0].inlineData.data;
      const sampleRate = parseInt(result.candidates[0].content.parts[0].inlineData.mimeType.split('rate=')[1]) || 24000;
      const binaryString = atob(audioData);
      const len = binaryString.length;
      const bytes = new Int16Array(len / 2);
      for (let i = 0; i < len; i += 2) bytes[i / 2] = (binaryString.charCodeAt(i + 1) << 8) | binaryString.charCodeAt(i);
      
      const wavHeader = (size, rate) => {
        const buf = new ArrayBuffer(44);
        const v = new DataView(buf);
        v.setUint32(0, 0x52494646); v.setUint32(4, 36 + size, true); v.setUint32(8, 0x57415645);
        v.setUint32(12, 0x666d7420); v.setUint32(16, 16, true); v.setUint16(20, 1, true);
        v.setUint16(22, 1, true); v.setUint32(24, rate, true); v.setUint32(28, rate * 2, true);
        v.setUint16(32, 2, true); v.setUint16(34, 16, true); v.setUint32(36, 0x64617461);
        v.setUint32(40, size, true);
        return buf;
      };
      const blob = new Blob([wavHeader(bytes.length * 2, sampleRate), bytes], { type: 'audio/wav' });
      const audio = new Audio(URL.createObjectURL(blob));
      audio.onended = () => {
        setIsSpeaking(false);
        setActiveSpeakerSection(null);
      };
      audio.play();
    } catch (e) { 
      setIsSpeaking(false); 
      setActiveSpeakerSection(null);
    }
  };

  const handleSendMessage = async () => {
    if (!inputMessage.trim()) return;
    const userMsg = { role: 'user', content: inputMessage };
    setMessages(prev => [...prev, userMsg]);
    setInputMessage("");

    const aiContent = await callAI(`User Query: ${inputMessage}`, `You are TracKademic Tutor for ${activeSubject?.name || 'General Study'}. Answer in precise grounding context.`, false);
    if (aiContent) {
      setMessages(prev => [...prev, { role: 'assistant', content: aiContent }]);
    }
  };

  // --- Source PDF Navigation & Zoom Upgrade ---
  const FullPDFViewer = () => {
    if (!pdfViewer.isOpen) return null;
    const [pages, setPages] = useState([]);
    const [loading, setLoading] = useState(true);
    const [pageJump, setPageJump] = useState("");
    const containerRef = useRef(null);

    useEffect(() => {
      const renderAll = async () => {
        if (!pdfViewer.file) return;
        setLoading(true);
        try {
          const pdf = await window.pdfjsLib.getDocument(pdfViewer.file).promise;
          const pageList = [];
          for (let i = 1; i <= pdf.numPages; i++) {
            const page = await pdf.getPage(i);
            const viewport = page.getViewport({ scale: pdfViewer.zoom });
            pageList.push({ page, viewport, num: i });
          }
          setPages(pageList);
          setLoading(false);
          
          setTimeout(() => {
            const target = document.getElementById(`pdf-page-${pdfViewer.page}`);
            if (target) target.scrollIntoView({ behavior: 'smooth', block: 'start' });
          }, 300);
        } catch (e) {
          addToast("Failed to render PDF pages.", "error");
          setLoading(false);
        }
      };
      renderAll();
    }, [pdfViewer.file, pdfViewer.zoom]);

    const handleJump = () => {
      const num = parseInt(pageJump);
      if (num > 0 && num <= pages.length) {
        const el = document.getElementById(`pdf-page-${num}`);
        if (el) el.scrollIntoView({ behavior: 'smooth', block: 'start' });
      } else {
        addToast("Invalid page number jump.", "error");
      }
    };

    const CanvasPage = ({ page, viewport, num }) => {
      const canvasRef = useRef();
      useEffect(() => {
        const ctx = canvasRef.current.getContext('2d');
        page.render({ canvasContext: ctx, viewport }).promise;
      }, [page, viewport]);
      return (
        <div id={`pdf-page-${num}`} className="mb-8 flex flex-col items-center">
          <div className="bg-white shadow-2xl rounded-2xl overflow-hidden border border-slate-200 transition-all">
            <canvas ref={canvasRef} width={viewport.width} height={viewport.height} />
          </div>
          <span className="mt-4 text-slate-400 font-bold text-xs uppercase tracking-widest">Page {num}</span>
        </div>
      );
    };

    return (
      <div className="fixed inset-0 z-[200] bg-slate-900/95 flex flex-col animate-in fade-in">
        <header className="p-6 border-b border-white/10 flex flex-col md:flex-row justify-between items-center gap-4 text-white shrink-0">
          <div className="flex items-center gap-4">
             <div className="p-2.5 bg-indigo-600 rounded-xl"><FileText size={20} /></div>
             <div>
               <h2 className="font-bold text-lg leading-none">TracKademic Reader</h2>
               <span className="text-xs text-slate-400 mt-1 block">Verified Source Material</span>
             </div>
          </div>

          <div className="flex flex-wrap items-center gap-3">
             <div className="flex items-center bg-slate-800 rounded-xl px-2 border border-slate-700">
               <input 
                 type="number" 
                 placeholder="Page" 
                 value={pageJump} 
                 onChange={(e) => setPageJump(e.target.value)}
                 className="bg-transparent text-sm w-16 p-2 outline-none border-none text-white font-medium"
               />
               <button onClick={handleJump} className="text-xs bg-indigo-600 text-white font-bold px-3 py-1.5 rounded-lg">Go</button>
             </div>

             <div className="flex bg-slate-800 rounded-xl border border-slate-700 overflow-hidden">
                <button onClick={() => setPdfViewer(prev => ({ ...prev, zoom: Math.max(0.75, prev.zoom - 0.25) }))} className="px-3 py-2 border-r border-slate-700 hover:bg-slate-700 font-bold text-sm">-</button>
                <span className="px-4 py-2 font-bold text-xs flex items-center text-slate-300">{Math.round(pdfViewer.zoom * 100)}%</span>
                <button onClick={() => setPdfViewer(prev => ({ ...prev, zoom: Math.min(2.0, prev.zoom + 0.25) }))} className="px-3 py-2 hover:bg-slate-700 font-bold text-sm">+</button>
             </div>

             <button onClick={() => setPdfViewer({ ...pdfViewer, isOpen: false })} className="p-3 bg-slate-800 hover:bg-slate-700 border border-slate-700 rounded-xl transition-colors"><X size={18} /></button>
          </div>
        </header>

        <div ref={containerRef} className="flex-1 overflow-y-auto p-10 flex flex-col items-center relative">
          {loading ? (
            <div className="h-full flex flex-col items-center justify-center gap-4">
              <div className="w-12 h-12 border-4 border-indigo-500/20 border-t-indigo-500 rounded-full animate-spin" />
              <p className="text-slate-400 font-bold text-sm">Rendering high-fidelity document nodes...</p>
            </div>
          ) : (
            pages.map(p => <CanvasPage key={p.num} {...p} />)
          )}
        </div>
      </div>
    );
  };

  // --- Render Toast Lists ---
  const ToastContainer = () => (
    <div className="fixed top-6 right-6 z-[300] flex flex-col gap-3 pointer-events-none">
      {toasts.map(t => (
        <div key={t.id} className={`p-4 rounded-2xl shadow-2xl border flex items-center gap-3 bg-white text-slate-800 animate-in slide-in-from-right-10 pointer-events-auto border-slate-100`}>
          <div className={`w-2.5 h-2.5 rounded-full ${t.type === 'error' ? 'bg-rose-500 animate-pulse' : t.type === 'info' ? 'bg-indigo-500' : 'bg-emerald-500'}`} />
          <p className="text-xs font-bold tracking-tight">{t.message}</p>
        </div>
      ))}
    </div>
  );

  // --- Schedule Plan Modal ---
  const StudyScheduleModal = () => {
    if (!scheduleModal.isOpen) return null;
    const plan = scheduleModal.plan;

    return (
      <div className="fixed inset-0 z-[210] bg-slate-900/80 backdrop-blur-md flex items-center justify-center p-6 animate-in fade-in">
        <div className="bg-white w-full max-w-4xl max-h-[90vh] rounded-[3rem] shadow-2xl flex flex-col overflow-hidden animate-in zoom-in-95">
          <header className="p-8 border-b border-slate-100 flex justify-between items-center bg-indigo-50/30">
            <div className="flex items-center gap-4">
               <div className="p-3 bg-indigo-600 text-white rounded-2xl shadow-lg"><Clock size={24} /></div>
               <div>
                  <h2 className="text-2xl font-black text-slate-900">Adaptive Curriculum Pacing Plan</h2>
                  <p className="text-xs text-indigo-600 font-bold uppercase tracking-widest mt-1">Grounded and Priority Weighted Pacing Strategy</p>
               </div>
            </div>
            <button onClick={() => setScheduleModal({ ...scheduleModal, isOpen: false })} className="p-2 hover:bg-slate-100 rounded-full"><X size={24} /></button>
          </header>
          
          <div className="flex-1 overflow-y-auto p-10 bg-white">
            {scheduleModal.isLoading ? (
              <div className="h-full flex flex-col items-center justify-center py-20">
                <div className="w-16 h-16 border-4 border-indigo-100 border-t-indigo-600 rounded-full animate-spin mb-6" />
                <p className="text-slate-400 font-bold">Optimizing study trajectory...</p>
              </div>
            ) : (
              <div className="space-y-8">
                <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
                  <div className="p-6 bg-slate-50 rounded-3xl border border-slate-100 flex flex-col justify-between">
                     <div>
                       <span className="text-[10px] font-black uppercase tracking-widest text-slate-400">Pacing Target</span>
                       <h3 className="text-2xl font-black text-slate-900 mt-2">Daily Commitment</h3>
                     </div>
                     <span className="text-4xl font-black text-indigo-600 mt-4">{plan?.recommendedHoursPerDay || 0} <span className="text-lg font-medium">Hours / Day</span></span>
                  </div>
                  <div className="p-6 bg-slate-50 rounded-3xl border border-slate-100 flex flex-col justify-between">
                     <div>
                       <span className="text-[10px] font-black uppercase tracking-widest text-slate-400">Target Date</span>
                       <h3 className="text-2xl font-black text-slate-900 mt-2">Estimated Completion</h3>
                     </div>
                     <span className="text-4xl font-black text-emerald-600 mt-4">{plan?.estimatedCompletionDate || 'N/A'}</span>
                  </div>
                </div>

                <div className="space-y-4">
                  <h3 className="text-sm font-black text-slate-400 uppercase tracking-widest">Day-by-Day Agenda</h3>
                  <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                     {plan?.studyPlan.map((d, i) => (
                       <div key={i} className="p-6 border border-slate-100 hover:border-indigo-200 hover:shadow-xl hover:shadow-indigo-500/5 rounded-3xl bg-white transition-all">
                          <div className="flex justify-between items-center mb-4">
                            <span className="px-3 py-1 bg-indigo-50 text-indigo-700 text-xs font-black rounded-lg">Day {d.day}</span>
                            <span className="text-xs font-bold text-slate-400">{d.hoursRequired} Hrs Estimated</span>
                          </div>
                          <h4 className="font-bold text-slate-800 text-base mb-3 leading-snug">Syllabus Targets:</h4>
                          <ul className="space-y-2 mb-4">
                             {d.topics.map((top, idx) => (
                               <li key={idx} className="text-xs font-semibold text-slate-600 flex items-center gap-2">
                                 <div className="w-1.5 h-1.5 bg-indigo-400 rounded-full shrink-0" /> {top}
                               </li>
                             ))}
                          </ul>
                          <p className="text-[11px] font-bold text-indigo-600/80 bg-indigo-50/50 p-2.5 rounded-xl">{d.focusMetric}</p>
                       </div>
                     ))}
                  </div>
                </div>

                <div className="p-6 bg-indigo-900 rounded-[2rem] text-white">
                  <h4 className="font-bold text-base mb-2 flex items-center gap-2"><Sparkles size={16} /> Strategy Advisor</h4>
                  <p className="text-xs text-indigo-100 leading-relaxed font-medium">{plan?.aiSummaryAdvice}</p>
                </div>
              </div>
            )}
          </div>
        </div>
      </div>
    );
  };

  const TopicCard = ({ topic, chapterId }) => {
    const score = getTopicScore(topic);
    const classification = getClassification(score);
    if (filterProb !== 'All' && topic.examProbability !== filterProb) return null;

    const openSourceDoc = (page) => {
      const fileData = pdfBinaryCache.get(activeSubject.id);
      if (fileData) {
        setPdfViewer({ isOpen: true, file: fileData, page: page, zoom: 1.0, searchTerm: "" });
      } else {
        addToast("Syllabus PDF cache is clear. Please upload PDF again.", "error");
      }
    };

    return (
      <div className="bg-white p-5 rounded-3xl border border-slate-100 hover:border-indigo-300 hover:shadow-xl hover:shadow-indigo-500/5 transition-all group/topic flex flex-col justify-between">
        <div className="flex justify-between items-start mb-4">
          <div className="flex-1 cursor-pointer" onClick={() => toggleTopic(activeSubject.id, chapterId, topic.id)}>
            <div className="flex items-center gap-3">
              <div className={`w-6 h-6 rounded-lg border-2 flex items-center justify-center transition-all ${topic.done ? 'bg-indigo-600 border-indigo-600' : 'border-slate-200 bg-white'}`}>
                {topic.done && <CheckCircle size={14} className="text-white" />}
              </div>
              <div>
                <span className={`text-base block ${topic.done ? 'text-slate-400 line-through font-medium' : 'text-slate-800 font-bold'}`}>{topic.name}</span>
                <div className="flex flex-wrap items-center gap-2 mt-2">
                   <span className={`text-[10px] font-black px-2 py-0.5 rounded-full border ${PROB_COLORS[topic.examProbability]}`}>
                     {topic.examProbability} Prob ({topic.probabilityScore || 0}%)
                   </span>
                   <span className={`text-[10px] font-black px-2 py-0.5 rounded-full border ${classification.color}`}>{classification.label}</span>
                </div>
              </div>
            </div>
          </div>
          <div className="flex gap-1">
             {topic.pastPaperFrequency > 0 && (
               <div className="flex items-center gap-1.5 px-2.5 py-1.5 bg-rose-50 text-rose-600 border border-rose-100 rounded-xl text-xs font-black animate-pulse">
                  <Flame size={12} fill="currentColor" /> {topic.pastPaperFrequency} Times
               </div>
             )}
          </div>
        </div>
        
        <div className="flex items-center justify-between pt-4 border-t border-slate-50 mt-4">
           <div className="flex gap-2">
              <button 
                onClick={() => openSourceDoc(topic.page)}
                className="p-2 text-slate-400 hover:text-indigo-600 hover:bg-indigo-50 rounded-lg transition-colors"
                title="View in PDF Source"
              >
                <ExternalLink size={18} />
              </button>
           </div>
           <button 
             onClick={() => explainTopicDetailed(topic)}
             className="px-4 py-2 bg-indigo-50 text-indigo-600 rounded-xl text-xs font-black hover:bg-indigo-600 hover:text-white transition-all flex items-center gap-1"
           >
              <Sparkles size={12} /> Detailed Notes
           </button>
        </div>
      </div>
    );
  };

  const SubjectDetail = () => {
    if (!activeSubject) return null;
    const remaining = activeSubject.chapters.reduce((acc, ch) => acc + ch.topics.filter(t => !t.done).length, 0);
    const total = activeSubject.chapters.reduce((acc, ch) => acc + ch.topics.length, 0);
    const progress = Math.round(( (total - remaining) / total ) * 100);

    return (
      <div className="p-8 max-w-7xl mx-auto animate-in slide-in-from-right duration-500 pb-32">
        <header className="flex flex-col md:flex-row justify-between items-start md:items-center mb-10 gap-6">
          <div className="flex items-center gap-6">
            <button onClick={() => setView('dashboard')} className="w-12 h-12 bg-white rounded-2xl flex items-center justify-center border border-slate-100 shadow-sm hover:bg-slate-50 transition-all">
              <ChevronRight className="rotate-180 text-slate-500" size={24} />
            </button>
            <div>
              <h1 className="text-4xl font-black text-slate-900 tracking-tight">{activeSubject.name}</h1>
              <div className="flex items-center gap-4 mt-2">
                 <span className="text-xs font-black bg-indigo-600 text-white px-3 py-1.5 rounded-full">{progress || 0}% Coverage</span>
                 <span className="text-xs font-bold text-slate-400 uppercase tracking-widest">{activeSubject.chapters.length} Chapters • {activeSubject.pastPapers.length} Papers</span>
              </div>
            </div>
          </div>
          <div className="flex gap-3">
             <button onClick={generateGuessPaper} className="px-6 py-3 bg-rose-600 text-white rounded-2xl font-black flex items-center gap-2 hover:bg-rose-700 shadow-xl shadow-rose-200 transition-all active:scale-95">
                <Flame size={18} fill="white" /> Generate Guess Paper
             </button>
          </div>
        </header>

        <div className="flex gap-1 p-1 bg-slate-100 rounded-2xl mb-10 w-fit">
           {['syllabus', 'pastpapers', 'predictions'].map(tab => (
             <button 
               key={tab}
               onClick={() => setActiveTab(tab)}
               className={`px-8 py-3 rounded-xl text-sm font-black transition-all capitalize ${activeTab === tab ? 'bg-white text-indigo-600 shadow-sm' : 'text-slate-500 hover:text-slate-800'}`}
             >
               {tab}
             </button>
           ))}
        </div>

        {activeTab === 'syllabus' && (
          <div className="grid grid-cols-1 lg:grid-cols-12 gap-10">
            <div className="lg:col-span-8 space-y-8">
               <div className="flex flex-col sm:flex-row justify-between items-start sm:items-center bg-white p-6 rounded-3xl border border-slate-100 shadow-sm gap-4">
                  <h3 className="font-black text-slate-800 flex items-center gap-3"><Filter size={18} className="text-slate-400" /> Filter Analysis</h3>
                  <div className="flex gap-2">
                     {['All', 'High', 'Medium', 'Low'].map(p => (
                       <button key={p} onClick={() => setFilterProb(p)} className={`px-4 py-2 rounded-xl text-xs font-bold border transition-all ${filterProb === p ? 'bg-slate-900 text-white border-slate-900' : 'bg-white text-slate-500 border-slate-100 hover:border-slate-300'}`}>{p}</button>
                     ))}
                  </div>
               </div>

               {activeSubject.chapters.map(chapter => (
                 <div key={chapter.id} className="space-y-4">
                    <div className="flex items-center justify-between px-4">
                       <h3 className="text-sm font-black text-slate-400 uppercase tracking-[0.2em]">{chapter.name}</h3>
                       <button 
                         onClick={() => {
                           setSubjects(prev => prev.map(s => s.id === activeSubjectId ? {
                             ...s, chapters: s.chapters.map(ch => ch.id === chapter.id ? {
                               ...ch, topics: ch.topics.map(t => ({ ...t, done: true }))
                             } : ch)
                           } : s));
                           addToast("All topics marked complete.", "success");
                         }}
                         className="text-xs font-bold text-indigo-600"
                       >
                         Mark all done
                       </button>
                    </div>
                    <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                       {chapter.topics.map(topic => <TopicCard key={topic.id} topic={topic} chapterId={chapter.id} />)}
                    </div>
                 </div>
               ))}
            </div>

            <div className="lg:col-span-4 space-y-8">
               <section className="bg-slate-900 text-white p-8 rounded-[2.5rem] shadow-2xl relative overflow-hidden">
                  <div className="relative z-10">
                     <h2 className="text-xl font-black mb-6 flex items-center gap-3"><TrendingUp className="text-indigo-400" /> Exam Trends</h2>
                     <div className="space-y-4">
                        <div className="p-5 bg-white/5 border border-white/10 rounded-2xl">
                           <p className="text-xs font-bold text-indigo-300 uppercase tracking-widest mb-1">Top Recurring Concept</p>
                           <p className="text-lg font-black truncate">{activeSubject.chapters.flatMap(c => c.topics).sort((a,b) => b.pastPaperFrequency - a.pastPaperFrequency)[0]?.name || 'N/A'}</p>
                        </div>
                        <div className="p-5 bg-white/5 border border-white/10 rounded-2xl">
                           <p className="text-xs font-bold text-indigo-300 uppercase tracking-widest mb-1">Topic Density</p>
                           <p className="text-lg font-black">{activeSubject.chapters.flatMap(c => c.topics).filter(t => t.examProbability === 'High').length} High Probability Areas</p>
                        </div>
                     </div>
                  </div>
                  <div className="absolute -bottom-10 -right-10 w-40 h-40 bg-indigo-500/20 rounded-full blur-3xl" />
               </section>

               <div className="bg-white p-8 rounded-[2.5rem] border border-slate-100 shadow-sm">
                  <h3 className="font-black text-slate-900 mb-6 flex items-center gap-3"><Clock size={20} className="text-indigo-600" /> Roadmap</h3>
                  <div className="space-y-4">
                     <div className="bg-slate-50 p-6 rounded-3xl text-center">
                        <div className="text-[10px] font-black uppercase tracking-widest text-slate-400 mb-1">Topics Left</div>
                        <div className="text-4xl font-black text-slate-900">{remaining}</div>
                     </div>
                     <button 
                       onClick={optimizeStudySchedule}
                       className="w-full py-4 bg-slate-900 text-white rounded-2xl font-black text-sm shadow-xl hover:bg-slate-800 transition-all active:scale-95 flex items-center justify-center gap-2"
                     >
                       <Target size={16} /> Optimize Study Schedule
                     </button>
                  </div>
               </div>
            </div>
          </div>
        )}

        {activeTab === 'pastpapers' && (
          <div className="space-y-10 animate-in fade-in">
             <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
                <label className="border-4 border-dashed border-slate-100 rounded-[2.5rem] flex flex-col items-center justify-center p-12 hover:bg-indigo-50 hover:border-indigo-200 cursor-pointer transition-all group text-center">
                   <input type="file" className="hidden" onChange={handlePastPaperUpload} accept=".pdf" />
                   <div className="w-16 h-16 bg-white rounded-3xl flex items-center justify-center shadow-md mb-4 group-hover:scale-110 transition-transform">
                      <Plus size={32} className="text-indigo-600" />
                   </div>
                   <span className="font-bold text-slate-400 group-hover:text-indigo-600">Upload Past Paper PDF</span>
                </label>

                {activeSubject.pastPapers.map(paper => (
                  <div key={paper.id} className="bg-white p-8 rounded-[2.5rem] border border-slate-100 shadow-sm flex flex-col items-center text-center">
                     <div className="w-16 h-16 bg-indigo-50 text-indigo-600 rounded-3xl flex items-center justify-center mb-6">
                        <FileQuestion size={32} />
                     </div>
                     <h3 className="text-lg font-bold text-slate-800 mb-2 truncate w-full px-4">{paper.name}</h3>
                     <p className="text-xs font-bold text-slate-400 uppercase tracking-widest mb-6">Analyzed & Mapped</p>
                     <button className="w-full py-3 bg-slate-50 text-slate-600 rounded-xl font-bold text-sm hover:bg-slate-100 transition-colors">View Details</button>
                  </div>
                ))}
             </div>
          </div>
        )}

        {activeTab === 'predictions' && (
          <div className="space-y-10 animate-in fade-in">
             <div className="bg-indigo-900 rounded-[3rem] p-12 text-white text-center relative overflow-hidden">
                <div className="relative z-10">
                   <h2 className="text-4xl font-black mb-4">Exam Ready?</h2>
                   <p className="text-lg text-indigo-200 max-w-2xl mx-auto mb-8">Our AI has analyzed your syllabus and uploaded past papers to generate a high-precision guess paper.</p>
                   <button onClick={generateGuessPaper} className="px-12 py-5 bg-white text-indigo-900 rounded-3xl font-black text-lg shadow-2xl hover:scale-105 transition-all active:scale-95">Generate Now</button>
                </div>
                <div className="absolute top-0 right-0 w-64 h-64 bg-white/5 rounded-full blur-3xl -mr-32 -mt-32" />
             </div>
          </div>
        )}
      </div>
    );
  };

  const Dashboard = () => {
    const overallProgress = getOverallProgress();
    
    // --- Dashboard Analytics Metrics ---
    const totalTopics = useMemo(() => subjects.reduce((acc, s) => acc + s.chapters.flatMap(c => c.topics).length, 0), [subjects]);
    const completedTopics = useMemo(() => subjects.reduce((acc, s) => acc + s.chapters.flatMap(c => c.topics).filter(t => t.done).length, 0), [subjects]);
    const remainingTopics = totalTopics - completedTopics;
    const highProbTopics = useMemo(() => subjects.reduce((acc, s) => acc + s.chapters.flatMap(c => c.topics).filter(t => t.examProbability === 'High').length, 0), [subjects]);
    const totalPastPapers = useMemo(() => subjects.reduce((acc, s) => acc + s.pastPapers.length, 0), [subjects]);

    const chartData = {
      labels: subjects.map(s => s.name),
      datasets: [{
        label: 'Completion %',
        data: subjects.map(s => calculateProgress(s)),
        backgroundColor: '#6366f1',
        borderRadius: 12,
      }]
    };

    return (
      <div className="p-8 space-y-12 max-w-7xl mx-auto animate-in fade-in">
        <header className="flex flex-col md:flex-row justify-between items-start md:items-center gap-6">
          <div>
            <h1 className="text-5xl font-black text-slate-900 tracking-tight">TracKademic</h1>
            <p className="text-slate-500 font-bold mt-2">The Exam Pattern Intelligence System.</p>
          </div>
          <div className="bg-white px-8 py-5 rounded-[2rem] shadow-sm border border-slate-100 flex items-center gap-6">
             <div className="w-14 h-14 bg-indigo-50 rounded-2xl flex items-center justify-center text-indigo-600"><Target size={32} /></div>
             <div>
                <div className="text-xs font-black text-slate-300 uppercase tracking-widest">Global Sync</div>
                <div className="text-3xl font-black">{overallProgress}%</div>
             </div>
          </div>
        </header>

        {/* Analytics Grid */}
        <div className="grid grid-cols-2 md:grid-cols-5 gap-4">
           {[
             { title: "Total Topics", value: totalTopics, color: "indigo" },
             { title: "Completed", value: completedTopics, color: "emerald" },
             { title: "Remaining", value: remainingTopics, color: "slate" },
             { title: "High Prob", value: highProbTopics, color: "rose" },
             { title: "Past Papers", value: totalPastPapers, color: "amber" }
           ].map((metric, i) => (
             <div key={i} className="bg-white p-5 rounded-3xl border border-slate-100 shadow-sm flex flex-col justify-between">
                <span className="text-[10px] font-black uppercase tracking-wider text-slate-400">{metric.title}</span>
                <span className="text-2xl font-black text-slate-800 mt-2">{metric.value}</span>
             </div>
           ))}
        </div>

        <div className="grid grid-cols-1 lg:grid-cols-3 gap-10">
          <div className="lg:col-span-2 space-y-10">
            <section className="bg-white p-10 rounded-[3rem] shadow-sm border border-slate-100">
               <h2 className="text-2xl font-black mb-10 flex items-center gap-4"><BarChart3 size={28} className="text-indigo-600" /> Syllabus Coverage Matrix</h2>
               <div className="h-72">
                  {subjects.length > 0 ? (
                    <Bar data={chartData} options={{ maintainAspectRatio: false, scales: { y: { beginAtZero: true, max: 100 } } }} />
                  ) : (
                    <div className="h-full flex flex-col items-center justify-center text-slate-300 border-2 border-dashed border-slate-100 rounded-[2rem]">
                      <Layers size={48} strokeWidth={1} className="mb-4" />
                      <p className="font-bold uppercase tracking-widest text-xs text-center px-4">No active curriculum indexes added</p>
                    </div>
                  )}
               </div>
            </section>

            <section className="grid grid-cols-1 md:grid-cols-2 gap-6">
              {subjects.map(s => (
                <div key={s.id} onClick={() => { setActiveSubjectId(s.id); setView('subject'); }} className="bg-white p-8 rounded-[2.5rem] border border-slate-100 shadow-sm hover:border-indigo-400 hover:shadow-2xl hover:shadow-indigo-500/10 transition-all cursor-pointer group">
                  <div className="flex justify-between items-start mb-6">
                    <div className="p-4 bg-indigo-600 text-white rounded-2xl shadow-lg shadow-indigo-100 transition-transform group-hover:scale-110"><BookOpen size={24} /></div>
                    <ShieldCheck size={24} className="text-emerald-500" />
                  </div>
                  <h3 className="text-2xl font-black text-slate-900 mb-2 truncate">{s.name}</h3>
                  <div className="flex items-center gap-3 text-xs font-bold text-slate-400 uppercase tracking-widest mb-6"><Calendar size={14} /> {s.deadline}</div>
                  <div className="w-full bg-slate-100 h-3 rounded-full overflow-hidden">
                    <div className="bg-indigo-600 h-full transition-all duration-1000" style={{ width: `${calculateProgress(s)}%` }} />
                  </div>
                </div>
              ))}
              <label className="border-4 border-dashed border-slate-100 rounded-[2.5rem] flex flex-col items-center justify-center p-12 hover:bg-indigo-50 hover:border-indigo-300 cursor-pointer transition-all group text-center">
                <input type="file" className="hidden" onChange={handleSyllabusUpload} accept=".pdf" />
                <div className="w-16 h-16 bg-white rounded-3xl flex items-center justify-center shadow-md mb-4 group-hover:scale-110 transition-transform"><Plus size={32} className="text-indigo-600" /></div>
                <span className="font-black text-slate-400 group-hover:text-indigo-600">Sync New Syllabus</span>
              </label>
            </section>
          </div>

          <div className="space-y-10">
            <section className="bg-slate-900 text-white p-10 rounded-[3rem] shadow-2xl relative overflow-hidden">
               <div className="relative z-10">
                  <h2 className="text-2xl font-black mb-8 flex items-center gap-4"><Zap className="text-yellow-400" /> AI Diagnostic Pacing</h2>
                  <div className="space-y-6">
                    {subjects.length > 0 ? (
                      <div className="p-6 bg-white/5 border border-white/10 rounded-3xl">
                        <p className="text-sm leading-relaxed text-slate-300">Active subjects optimized. Grounded explanation modal configured for math and circuit diagrams.</p>
                      </div>
                    ) : (
                      <p className="text-sm opacity-50 italic">Telemetry analyzer offline. Sync a curriculum PDF to begin diagnosis.</p>
                    )}
                  </div>
               </div>
               <div className="absolute -top-10 -right-10 w-48 h-48 bg-indigo-500/20 rounded-full blur-3xl" />
            </section>
          </div>
        </div>
      </div>
    );
  };

  // --- Presentation-Style Learning Notes Modal Component ---
  const AIContentModal = () => {
    if (!aiModal.isOpen) return null;
    const data = aiModal.content;

    return (
      <div className="fixed inset-0 z-[100] flex items-center justify-center p-6 bg-slate-900/70 backdrop-blur-md animate-in fade-in duration-300">
        <div className="bg-white w-full max-w-4xl max-h-[85vh] rounded-[3rem] shadow-2xl overflow-hidden flex flex-col animate-in zoom-in-95 duration-200">
          <header className="p-8 border-b border-slate-100 flex justify-between items-center bg-indigo-50/30 shrink-0">
            <div className="flex items-center gap-4">
              <div className="p-3 bg-indigo-600 text-white rounded-2xl shadow-lg"><BrainCircuit size={28} /></div>
              <div>
                <h2 className="text-2xl font-black text-slate-800 tracking-tight">{aiModal.title}</h2>
                <div className="flex items-center gap-2 text-xs font-bold text-emerald-600 uppercase tracking-widest mt-1">
                  <ShieldCheck size={14} /> TracKademic Presentation-Style Learning Notes
                </div>
              </div>
            </div>
            <button onClick={() => {
              setAiModal({ ...aiModal, isOpen: false });
              setIsSpeaking(false);
              setActiveSpeakerSection(null);
            }} className="p-2 hover:bg-slate-200 rounded-full transition-colors"><X size={28} /></button>
          </header>
          
          <div className="flex-1 overflow-y-auto p-12 bg-white space-y-10">
            {aiModal.isLoading ? (
              <div className="flex flex-col items-center justify-center py-20 gap-6">
                <div className="w-16 h-16 border-4 border-indigo-100 border-t-indigo-600 rounded-full animate-spin" />
                <p className="text-slate-400 font-black uppercase text-xs tracking-widest">Compiling source materials into structural lecture decks...</p>
              </div>
            ) : data ? (
              <div className="space-y-10 animate-in fade-in duration-300">
                {/* Section 1: Gradient Summary Hero Card */}
                <div className="bg-gradient-to-r from-indigo-900 to-indigo-950 p-8 rounded-[2rem] text-white relative overflow-hidden shadow-xl">
                  <div className="relative z-10 flex justify-between items-start">
                     <div>
                       <span className="text-[10px] font-black uppercase tracking-widest text-indigo-300">Subject Overview</span>
                       <h3 className="text-2xl font-bold mt-2 mb-4 leading-snug">{data.title}</h3>
                       <p className="text-sm text-indigo-100 leading-relaxed font-medium">{data.summary}</p>
                     </div>
                     <button 
                       onClick={() => speakTextGrounded(data.summary, 'overview')}
                       className="p-3 bg-white/10 hover:bg-white/20 rounded-2xl transition-all"
                     >
                       <Volume2 className={activeSpeakerSection === 'overview' ? "animate-bounce text-emerald-300" : "text-white"} size={20} />
                     </button>
                  </div>
                </div>

                {/* Section 2: Key Points Grid */}
                <div className="space-y-4">
                  <h3 className="text-xs font-black text-slate-400 uppercase tracking-widest flex items-center justify-between">
                     <span>Key Conceptual Insights</span>
                     <button onClick={() => speakTextGrounded(data.keyPoints.join(' '), 'keyPoints')} className="p-1.5 hover:bg-slate-50 rounded-lg text-indigo-600"><Volume2 size={16} /></button>
                  </h3>
                  <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                     {data.keyPoints?.map((p, i) => (
                       <div key={i} className="p-6 bg-slate-50 border border-slate-100 rounded-3xl flex gap-4 items-start">
                          <span className="text-xs font-bold bg-indigo-100 text-indigo-700 px-2.5 py-1 rounded-xl shrink-0">0{i+1}</span>
                          <p className="text-sm font-semibold text-slate-600 leading-relaxed">{p}</p>
                       </div>
                     ))}
                  </div>
                </div>

                {/* Section 3: Important Definitions & Laws */}
                <div className="grid grid-cols-1 md:grid-cols-2 gap-8">
                  {data.importantDefinitions && data.importantDefinitions.length > 0 && (
                    <div className="space-y-4">
                       <h3 className="text-xs font-black text-slate-400 uppercase tracking-widest flex items-center justify-between">
                         <span>Important Definitions</span>
                         <button onClick={() => speakTextGrounded(data.importantDefinitions.map(d=> `${d.term}: ${d.definition}`).join(' '), 'defs')} className="p-1.5 hover:bg-slate-50 rounded-lg text-indigo-600"><Volume2 size={16} /></button>
                       </h3>
                       <div className="space-y-3">
                          {data.importantDefinitions.map((d, i) => (
                             <div key={i} className="p-5 border border-l-4 border-indigo-600 bg-indigo-50/20 rounded-2xl">
                                <h4 className="font-bold text-indigo-900 mb-1">{d.term}</h4>
                                <p className="text-xs font-semibold text-slate-600 leading-relaxed">{d.definition}</p>
                             </div>
                          ))}
                       </div>
                    </div>
                  )}

                  {/* Formulas Section */}
                  {data.formulas && data.formulas.length > 0 && (
                    <div className="space-y-4">
                       <h3 className="text-xs font-black text-slate-400 uppercase tracking-widest">Mathematical Equations</h3>
                       <div className="space-y-3">
                          {data.formulas.map((f, i) => (
                             <div key={i} className="p-6 bg-slate-50 border border-slate-100 rounded-2xl flex flex-col justify-center items-center gap-3">
                                <span className="text-[10px] font-black uppercase text-slate-400 tracking-wider">{f.name}</span>
                                <div className="text-xl font-bold bg-white px-6 py-4 rounded-xl border border-slate-200">
                                   <FormulaRenderer text={`$$${f.latex}$$`} />
                                </div>
                             </div>
                          ))}
                       </div>
                    </div>
                  )}
                </div>

                {/* Section 4: Memory Flashcard Tricks Section */}
                {data.memoryTricks && data.memoryTricks.length > 0 && (
                  <div className="space-y-4">
                     <h3 className="text-xs font-black text-slate-400 uppercase tracking-widest flex items-center justify-between">
                       <span>Flashcard Mnemonic Tricks</span>
                       <button onClick={() => speakTextGrounded(data.memoryTricks.map(t=> `Concept: ${t.concept}. Memory Trick: ${t.trick}`).join(' '), 'tricks')} className="p-1.5 hover:bg-slate-50 rounded-lg text-indigo-600"><Volume2 size={16} /></button>
                     </h3>
                     <div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
                        {data.memoryTricks.map((t, i) => (
                           <div key={i} className="bg-slate-50 hover:bg-indigo-50 border border-slate-100 hover:border-indigo-100 rounded-3xl p-6 transition-all shadow-sm">
                              <span className="text-[10px] font-black uppercase tracking-wider text-slate-400">{t.concept}</span>
                              <p className="text-sm font-black text-indigo-900 mt-2 leading-snug">{t.trick}</p>
                           </div>
                        ))}
                     </div>
                  </div>
                )}

                {/* Section 5: Exam Warnings & Tips */}
                {data.examTips && data.examTips.length > 0 && (
                  <div className="space-y-4">
                     <h3 className="text-xs font-black text-slate-400 uppercase tracking-widest">Critical Exam Strategies</h3>
                     <div className="space-y-3">
                        {data.examTips.map((t, i) => (
                           <div key={i} className="p-5 bg-amber-50 border border-amber-100 text-amber-900 rounded-2xl flex gap-3 items-start">
                              <AlertCircle className="shrink-0 text-amber-600 mt-0.5" size={18} />
                              <p className="text-xs font-bold leading-relaxed">{t}</p>
                           </div>
                        ))}
                     </div>
                  </div>
                )}

                {/* Section 6: Past Paper Questions accordion */}
                {data.commonQuestions && data.commonQuestions.length > 0 && (
                  <div className="space-y-4">
                     <h3 className="text-xs font-black text-slate-400 uppercase tracking-widest">Direct Sample Exam Questions</h3>
                     <div className="space-y-3">
                        {data.commonQuestions.map((q, i) => (
                           <div key={i} className="p-6 border border-slate-100 rounded-3xl bg-white shadow-sm">
                              <div className="flex justify-between items-center mb-3">
                                <span className="text-xs font-black text-slate-400 uppercase tracking-widest">Question {i+1}</span>
                                <span className="px-3 py-1 bg-rose-50 text-rose-600 border border-rose-100 text-xs font-black rounded-lg">{q.marks} Marks</span>
                              </div>
                              <p className="font-bold text-slate-800 text-base mb-3">{q.question}</p>
                              <div className="p-4 bg-slate-50 border border-slate-100 rounded-2xl text-xs font-semibold leading-relaxed text-slate-600">
                                 <strong className="block mb-1 text-slate-800 uppercase tracking-wider text-[10px] font-black">Evaluation Criteria:</strong>
                                 {q.answer}
                              </div>
                           </div>
                        ))}
                     </div>
                  </div>
                )}
              </div>
            ) : (
              <div className="text-center py-20 text-slate-400">Failed to render parsed AI dataset.</div>
            )}
          </div>
        </div>
      </div>
    );
  };

  // --- Upgraded Guess Paper Modal with Real PDF Exporter ---
  const GuessPaperModal = () => {
    if (!guessPaper.isOpen) return null;
    return (
      <div className="fixed inset-0 z-[210] bg-slate-900/80 backdrop-blur-md flex items-center justify-center p-6 animate-in fade-in">
        <div className="bg-white w-full max-w-4xl max-h-[90vh] rounded-[3rem] shadow-2xl flex flex-col overflow-hidden animate-in zoom-in-95">
          <header className="p-8 border-b border-slate-100 flex justify-between items-center bg-rose-50/30 shrink-0">
             <div className="flex items-center gap-4">
                <div className="p-3 bg-rose-600 text-white rounded-2xl shadow-lg"><Flame size={24} /></div>
                <div>
                   <h2 className="text-2xl font-black text-slate-900 font-sans">Predicted Syllabus Guess Paper</h2>
                   <p className="text-xs text-rose-600 font-bold uppercase tracking-widest mt-1">Cross-Referenced Exam Probability Weight</p>
                </div>
             </div>
             <button onClick={() => setGuessPaper({ ...guessPaper, isOpen: false })} className="p-2 hover:bg-slate-100 rounded-full"><X size={24} /></button>
          </header>
          
          <div className="flex-1 overflow-y-auto p-12 bg-white" id="guess-paper-printable">
             {guessPaper.isLoading ? (
               <div className="h-full flex flex-col items-center justify-center py-20">
                  <div className="w-16 h-16 border-4 border-rose-100 border-t-rose-600 rounded-full animate-spin mb-6" />
                  <p className="text-slate-400 font-bold">Assembling predicted exam questions...</p>
               </div>
             ) : (
               <div className="space-y-12">
                  <div className="p-6 bg-slate-50 border border-slate-100 rounded-3xl">
                     <span className="text-[10px] font-black uppercase text-slate-400 tracking-wider">Exam Scheme</span>
                     <h3 className="text-2xl font-black text-slate-900 mt-1">{activeSubject?.name} • Mock Predict</h3>
                     <span className="text-xs text-slate-400 mt-1 block">Compiled on: {new Date().toLocaleDateString()}</span>
                  </div>

                  <section>
                     <h3 className="text-sm font-black text-slate-400 uppercase tracking-widest mb-6 flex items-center gap-3">
                        <div className="w-1.5 h-1.5 rounded-full bg-rose-500" /> Section A: MCQs
                     </h3>
                     <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                        {guessPaper.content?.mcqs.map((q, i) => (
                           <div key={i} className="p-5 bg-slate-50 border border-slate-100 rounded-2xl text-sm font-semibold text-slate-700">{q}</div>
                        ))}
                     </div>
                  </section>
                  <section>
                     <h3 className="text-sm font-black text-slate-400 uppercase tracking-widest mb-6 flex items-center gap-3">
                        <div className="w-1.5 h-1.5 rounded-full bg-rose-500" /> Section B: Short Questions
                     </h3>
                     <div className="space-y-3">
                        {guessPaper.content?.shortQuestions.map((q, i) => (
                           <div key={i} className="p-5 border border-slate-100 rounded-2xl text-sm font-semibold text-slate-700">{q}</div>
                        ))}
                     </div>
                  </section>
                  <section>
                     <h3 className="text-sm font-black text-slate-400 uppercase tracking-widest mb-6 flex items-center gap-3">
                        <div className="w-1.5 h-1.5 rounded-full bg-rose-500" /> Section C: Descriptive Questions
                     </h3>
                     <div className="space-y-3">
                        {guessPaper.content?.longQuestions.map((q, i) => (
                           <div key={i} className="p-6 bg-slate-900 text-slate-200 rounded-3xl text-base font-bold shadow-xl border border-white/5">{q}</div>
                        ))}
                     </div>
                  </section>
               </div>
             )}
          </div>
          <footer className="p-8 border-t border-slate-100 flex justify-end gap-3 bg-slate-50/50 shrink-0">
             <button onClick={downloadGuessPaperPdf} className="px-6 py-3 bg-white border border-slate-200 rounded-2xl font-bold flex items-center gap-2 hover:bg-slate-50 transition-all"><Download size={18} /> Download Guess Paper PDF</button>
             <button onClick={() => setGuessPaper({ ...guessPaper, isOpen: false })} className="px-8 py-3 bg-rose-600 text-white rounded-2xl font-bold shadow-lg shadow-rose-200 hover:bg-rose-700">Close Predictor</button>
          </footer>
        </div>
      </div>
    );
  };

  return (
    <div className="min-h-screen bg-[#F8FAFC] flex font-sans text-slate-900 selection:bg-indigo-100">
      <AIContentModal />
      <FullPDFViewer />
      <GuessPaperModal />
      <StudyScheduleModal />
      <ToastContainer />
      
      <aside className="w-80 bg-white border-r border-slate-200 hidden lg:flex flex-col shrink-0 sticky top-0 h-screen">
        <div className="p-10 flex-1 flex flex-col">
          <div className="flex items-center gap-3 text-indigo-600 mb-16">
            <div className="w-14 h-14 bg-indigo-600 rounded-2xl flex items-center justify-center text-white shadow-xl shadow-indigo-100"><Zap size={32} fill="white" /></div>
            <span className="text-3xl font-black text-slate-900 tracking-tighter">TracKademic</span>
          </div>
          <nav className="space-y-3 flex-1">
            <button onClick={() => setView('dashboard')} className={`w-full flex items-center gap-4 px-6 py-5 rounded-3xl transition-all ${view === 'dashboard' ? 'bg-slate-900 text-white shadow-2xl' : 'text-slate-500 hover:bg-slate-50 font-bold'}`}><BarChart3 size={24} /> Dashboard</button>
            <div className="pt-12 pb-4 text-[11px] font-black text-slate-300 uppercase tracking-[0.2em] px-6">Source Analytics</div>
            <div className="space-y-2 max-h-[40vh] overflow-y-auto px-2 custom-scrollbar">
              {subjects.map(s => (
                <button key={s.id} onClick={() => { setActiveSubjectId(s.id); setView('subject'); }} className={`w-full flex items-center gap-4 px-6 py-4 rounded-2xl transition-all ${activeSubjectId === s.id && view === 'subject' ? 'bg-indigo-50 text-indigo-700 font-black shadow-sm' : 'text-slate-500 hover:bg-slate-50'}`}>
                  <div className={`w-3 h-3 rounded-full ${calculateProgress(s) === 100 ? 'bg-emerald-500' : 'bg-indigo-300'}`} />
                  <span className="truncate text-sm">{s.name}</span>
                </button>
              ))}
            </div>
          </nav>
          <label className="mt-auto flex items-center justify-center gap-4 w-full py-6 bg-indigo-600 text-white rounded-[2rem] font-black cursor-pointer hover:bg-indigo-700 transition-all shadow-2xl shadow-indigo-100 group">
             <Upload size={24} className="group-hover:-translate-y-1 transition-transform" />
             <span>Sync Syllabus</span>
             <input type="file" className="hidden" onChange={handleSyllabusUpload} accept=".pdf" />
          </label>
        </div>
      </aside>

      <main className="flex-1 relative">
         {isAnalyzing && (
            <div className="absolute inset-0 z-[300] bg-white/95 backdrop-blur-xl flex flex-col items-center justify-center animate-in fade-in">
               <div className="w-24 h-24 border-[6px] border-slate-100 border-t-indigo-600 rounded-full animate-spin mb-8 shadow-inner" />
               <h2 className="text-3xl font-black text-slate-900">Pattern Recognition Engine</h2>
               <p className="text-slate-500 mt-2 font-medium tracking-wide">Mapping syllabus topics to past paper trends...</p>
            </div>
         )}
         {view === 'dashboard' ? <Dashboard /> : <SubjectDetail />}
      </main>
    </div>
  );
};

export default App;
