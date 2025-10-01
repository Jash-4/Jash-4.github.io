# Jash.github.io

import React, { useState, useEffect, useRef, useMemo, useCallback } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, onSnapshot, collection, query, setDoc, updateDoc, arrayUnion, deleteDoc, getDocs, writeBatch } from 'firebase/firestore';
import { Calendar, ChevronLeft, ChevronRight, Folder, Clock, CheckCircle, Award, BarChart3, Users, X, AlertTriangle, List, RefreshCcw, Trash2, Bell, Edit2, Zap } from 'lucide-react';

// --- Global Variables (Provided by Canvas Environment) ---
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : null;
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';

// --- Firebase Initialization (Simulated if not in Canvas) ---
let app, db, auth;
if (firebaseConfig) {
    app = initializeApp(firebaseConfig);
    db = getFirestore(app);
    auth = getAuth(app);
}

// Helper to get firestore path
const getSubjectCollectionRef = (userId) => collection(db, `artifacts/${appId}/users/${userId}/subjects`);
const getTimeLogCollectionRef = (userId) => collection(db, `artifacts/${appId}/users/${userId}/time_logs`);
const getGradesCollectionRef = (userId) => collection(db, `artifacts/${appId}/users/${userId}/grades`);
const getCreditDocRef = (userId) => doc(db, `artifacts/${appId}/users/${userId}/credits`, 'balance');

// Utility Functions
const formatDate = (date) => {
    const d = new Date(date);
    const year = d.getFullYear();
    const month = String(d.getMonth() + 1).padStart(2, '0');
    const day = String(d.getDate()).padStart(2, '0');
    return `${year}-${month}-${day}`;
};

// Available colors for custom selection
const COLOR_OPTIONS = [
    { name: 'None (Auto Priority)', value: '' },
    { name: 'Red', value: 'red' },
    { name: 'Orange', value: 'orange' },
    { name: 'Yellow', value: 'yellow' },
    { name: 'Green', value: 'green' },
    { name: 'Blue', value: 'blue' },
    { name: 'Indigo', value: 'indigo' },
    { name: 'Pink', value: 'pink' },
];

// --- Web Audio API functions ---

// Context is created outside the component to avoid re-creation issues
const audioContext = new (window.AudioContext || window.webkitAudioContext)();

// Function to generate the sound using Web Audio API
const generateSound = (type) => {
    if (audioContext.state === 'suspended') {
        audioContext.resume();
    }
    
    // Play sound 3 times
    for (let i = 0; i < 3; i++) {
        setTimeout(() => {
            if (type === 'chime') {
                playChime();
            } else {
                playHighBeep();
            }
        }, i * 200); // 200ms delay between beeps
    }
};

const playHighBeep = () => {
    const oscillator = audioContext.createOscillator();
    const gainNode = audioContext.createGain();

    oscillator.connect(gainNode);
    gainNode.connect(audioContext.destination);

    oscillator.type = 'triangle';
    oscillator.frequency.setValueAtTime(880, audioContext.currentTime); // High frequency

    gainNode.gain.setValueAtTime(0.5, audioContext.currentTime);
    gainNode.gain.exponentialRampToValueAtTime(0.001, audioContext.currentTime + 0.15); // Quick fade

    oscillator.start(audioContext.currentTime);
    oscillator.stop(audioContext.currentTime + 0.15);
};

const playChime = () => {
    const notes = [
        { freq: 440, duration: 0.2 }, // A4
        { freq: 554, duration: 0.4 }, // C#5
    ];

    notes.forEach((note, index) => {
        const oscillator = audioContext.createOscillator();
        const gainNode = audioContext.createGain();

        oscillator.connect(gainNode);
        gainNode.connect(audioContext.destination);

        oscillator.type = 'sine';
        oscillator.frequency.setValueAtTime(note.freq, audioContext.currentTime + index * 0.1);

        gainNode.gain.setValueAtTime(0.5, audioContext.currentTime + index * 0.1);
        gainNode.gain.exponentialRampToValueAtTime(0.001, audioContext.currentTime + index * 0.1 + note.duration);

        oscillator.start(audioContext.currentTime + index * 0.1);
        oscillator.stop(audioContext.currentTime + index * 0.1 + note.duration);
    });
};


// Extracted Grade Form Component for stability 
const GradeFormComponent = React.memo(({ newGrade, handleGradeChange, handleSubmitGrade, subjectOptions }) => (
    <form onSubmit={handleSubmitGrade} className="space-y-4">
        <div>
            <label className="block text-sm font-medium text-gray-700">Subject</label>
            <select
                name="subjectId"
                value={newGrade.subjectId}
                onChange={handleGradeChange}
                className="w-full p-2 border rounded-lg"
                required
            >
                <option value="">Select Subject</option>
                {subjectOptions.map(option => (
                    <option key={option.value} value={option.value}>{option.label}</option>
                ))}
            </select>
        </div>
        <div>
            <label className="block text-sm font-medium text-gray-700">Date of Test</label>
            <input type="date" name="date" value={newGrade.date} onChange={handleGradeChange} className="w-full p-2 border rounded-lg" required />
        </div>
        <div>
            <label className="block text-sm font-medium text-gray-700">Type of Exam</label>
            <input 
                type="text" 
                name="type" 
                value={newGrade.type} 
                onChange={handleGradeChange} 
                placeholder="e.g., Midterm, Quiz, Final" 
                className="w-full p-2 border rounded-lg" 
                required 
            />
        </div>
        <div className="flex gap-4">
            <div className="flex-1">
                <label className="block text-sm font-medium text-gray-700">Marks Obtained</label>
                <input type="number" name="marksObtained" value={newGrade.marksObtained} onChange={handleGradeChange} placeholder="e.g., 85" className="w-full p-2 border rounded-lg" required min="0" />
            </div>
            <div className="flex-1">
                <label className="block text-sm font-medium text-gray-700">Total Marks</label>
                <input type="number" name="totalMarks" value={newGrade.totalMarks} onChange={handleGradeChange} placeholder="e.g., 100" className="w-full p-2 border rounded-lg" required min="1" />
            </div>
        </div>
        <button type="submit" className="w-full py-2 bg-indigo-600 text-white font-semibold rounded-lg hover:bg-indigo-700 transition-colors">
            Save Grade
        </button>
    </form>
));

const App = () => {
    const [view, setView] = useState('tasks'); // 'tasks', 'timer', 'calendar', 'grades', 'history'
    const [userId, setUserId] = useState(null);
    const [isAuthReady, setIsAuthReady] = useState(false);
    const [subjects, setSubjects] = useState([]);
    const [timeLogs, setTimeLogs] = useState([]);
    const [grades, setGrades] = useState([]);
    const [credit, setCredit] = useState({ balance: 0, history: [] });
    const [currentSubjectId, setCurrentSubjectId] = useState(null);
    const [errorMessage, setErrorMessage] = useState('');
    const [notification, setNotification] = useState(null); // For custom timer reminders

    // --- Audio Control State ---
    const [soundStyle, setSoundStyle] = useState('highBeep'); // 'highBeep' or 'chime'
    const [isAudioInitialized, setIsAudioInitialized] = useState(false); 
    
    // --- Modals State ---
    const [showDeleteLogModal, setShowDeleteLogModal] = useState(false);
    const [showResetBalanceModal, setShowResetBalanceModal] = useState(false);
    const [showEditSubjectModal, setShowEditSubjectModal] = useState(null); // subject data
    const [showEditFolderModal, setShowEditFolderModal] = useState(null); // {subjectId, folder}
    const [showEditTaskModal, setShowEditTaskModal] = useState(null); // {subjectId, task}


    const playReminderSound = () => {
        // Only generate sound if initialization permission has been granted
        if (isAudioInitialized) {
            generateSound(soundStyle); 
        }
    };

    const testReminderSound = () => {
        // CRITICAL: This explicit user gesture allows the audio context to start
        if (audioContext.state === 'suspended') {
            audioContext.resume();
        }
        setIsAudioInitialized(true);
        generateSound(soundStyle);
    }
    // --- End Audio Control ---

    // --- Authentication and Firestore Setup ---
    useEffect(() => {
        if (!db || !auth) {
            setErrorMessage("Firebase is not configured. Data will not persist.");
            setIsAuthReady(true);
            return;
        }

        const setupAuth = async () => {
            try {
                if (initialAuthToken) {
                    await signInWithCustomToken(auth, initialAuthToken);
                } else {
                    await signInAnonymously(auth);
                }
            } catch (error) {
                console.error("Auth error:", error);
                setErrorMessage("Failed to sign in. Data persistence disabled.");
            }
        };

        const unsubscribe = onAuthStateChanged(auth, (user) => {
            if (user) {
                setUserId(user.uid);
            } else {
                setupAuth(); // If no user, attempt sign in
            }
            setIsAuthReady(true);
        });

        return () => unsubscribe();
    }, []);

    // --- Realtime Data Fetching ---
    useEffect(() => {
        if (!isAuthReady || !userId || !db) return;

        const unsubSubjects = onSnapshot(getSubjectCollectionRef(userId), (snapshot) => {
            setSubjects(snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() })));
        }, (error) => console.error("Subjects snapshot error:", error));

        const unsubTimeLogs = onSnapshot(getTimeLogCollectionRef(userId), (snapshot) => {
            setTimeLogs(snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() })));
        }, (error) => console.error("TimeLogs snapshot error:", error));

        const unsubGrades = onSnapshot(getGradesCollectionRef(userId), (snapshot) => {
            setGrades(snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() })));
        }, (error) => console.error("Grades snapshot error:", error));

        const unsubCredit = onSnapshot(getCreditDocRef(userId), (doc) => {
            if (doc.exists()) {
                setCredit({ balance: doc.data().balance || 0, history: doc.data().history || [] });
                checkAndResetMonthlyCredit(doc.data().lastReset);
            } else {
                setDoc(getCreditDocRef(userId), { balance: 0, history: [], lastReset: new Date() });
            }
        }, (error) => console.error("Credit snapshot error:", error));

        return () => {
            unsubSubjects();
            unsubTimeLogs();
            unsubGrades();
            unsubCredit();
        };
    }, [isAuthReady, userId]);

    // --- Credit Reset Logic ---
    const handleManualBalanceReset = async () => {
        if (!userId || !db) return;

        const now = new Date();
        const previousBalance = credit.balance;

        try {
            await updateDoc(getCreditDocRef(userId), {
                balance: 0,
                history: arrayUnion({
                    id: crypto.randomUUID(),
                    type: 'Manual Reset',
                    amount: -previousBalance,
                    date: now.toISOString(),
                    note: `Manual balance reset: $${previousBalance.toFixed(2)} zeroed out.`,
                }),
            });
            setShowResetBalanceModal(false);
        } catch (e) {
            console.error("Error resetting balance:", e);
            setErrorMessage("Failed to reset balance.");
        }
    };


    const checkAndResetMonthlyCredit = useCallback(async (lastReset) => {
        if (!userId || !db) return;

        const now = new Date();
        const lastResetDate = lastReset instanceof Date ? lastReset : (lastReset && lastReset.toDate ? lastReset.toDate() : new Date(0));
        
        // Check if the month has changed since the last reset
        if (now.getMonth() !== lastResetDate.getMonth() || now.getFullYear() !== lastResetDate.getFullYear()) {
            console.log("Monthly credit reset triggered.");
            const previousBalance = credit.balance;

            if (previousBalance > 0) {
                 await updateDoc(getCreditDocRef(userId), {
                    balance: 0,
                    lastReset: now,
                    history: arrayUnion({
                        id: crypto.randomUUID(), // Unique ID for history item
                        type: 'Monthly Reset',
                        amount: -previousBalance,
                        date: now.toISOString(),
                        note: `Previous month's balance of $${previousBalance.toFixed(2)} was transferred/zeroed out.`
                    })
                });
            } else {
                await updateDoc(getCreditDocRef(userId), { lastReset: now });
            }
        }
    }, [userId, credit.balance]);

    // --- Task Management Handlers ---

    // 1. Add Subject/Main Folder
    const handleAddSubject = async (subjectName) => {
        if (!userId || !db || !subjectName) return;
        try {
            await setDoc(doc(getSubjectCollectionRef(userId)), {
                name: subjectName,
                folders: [],
                tasks: [],
            });
        } catch (e) { console.error("Error adding subject:", e); }
    };
    
    // 1b. Edit Subject
    const handleEditSubject = async (subjectId, newName) => {
        if (!userId || !db || !newName) return;
        const subjectDocRef = doc(db, `artifacts/${appId}/users/${userId}/subjects`, subjectId);
        try {
            await updateDoc(subjectDocRef, { name: newName });
            setShowEditSubjectModal(null);
        } catch (e) { console.error("Error editing subject:", e); }
    }


    // 2. Add Sub-Folder
    const handleAddFolder = async (subjectId, folder) => {
        if (!userId || !db || !folder.name) return;
        const subjectDocRef = doc(db, `artifacts/${appId}/users/${userId}/subjects`, subjectId);
        
        const newFolder = {
            id: crypto.randomUUID(), // Use UUID for unique folder ID
            name: folder.name,
            description: folder.description || '',
            color: folder.color || '',
        };

        try {
            await updateDoc(subjectDocRef, {
                folders: arrayUnion(newFolder),
            });
        } catch (e) { console.error("Error adding folder:", e); }
    };
    
    // 2b. Edit Sub-Folder
    const handleEditFolder = async (subjectId, updatedFolder) => {
        if (!userId || !db) return;
        const subjectDocRef = doc(db, `artifacts/${appId}/users/${userId}/subjects`, subjectId);
        const subjectData = subjects.find(s => s.id === subjectId);
        if (!subjectData) return;

        const updatedFolders = subjectData.folders.map(f => f.id === updatedFolder.id ? updatedFolder : f);

        try {
            await updateDoc(subjectDocRef, { folders: updatedFolders });
            setShowEditFolderModal(null);
        } catch (e) { console.error("Error editing folder:", e); }
    }


    // 3. Add Task
    const handleAddTask = async (subjectId, folderId, task) => {
        if (!userId || !db) return;
        const subjectDocRef = doc(db, `artifacts/${appId}/users/${userId}/subjects`, subjectId);
        const newTask = {
            id: crypto.randomUUID(),
            folderId: folderId || null,
            text: task.text,
            description: task.description || '',
            deadline: task.deadline,
            credits: parseFloat(task.credits) || 0,
            color: task.color || '', // New color field
            isCompleted: false,
            isStrikethrough: false,
            createdAt: new Date().toISOString(),
        };

        try {
            await updateDoc(subjectDocRef, {
                tasks: arrayUnion(newTask)
            });
        } catch (e) { console.error("Error adding task:", e); }
    };
    
    // 3b. Edit Task
    const handleEditTask = async (subjectId, updatedTask) => {
        if (!userId || !db) return;
        const subjectDocRef = doc(db, `artifacts/${appId}/users/${userId}/subjects`, subjectId);
        const subjectData = subjects.find(s => s.id === subjectId);
        if (!subjectData) return;
        
        const updatedTasks = subjectData.tasks.map(t => t.id === updatedTask.id ? updatedTask : t);

        try {
            await updateDoc(subjectDocRef, { tasks: updatedTasks });
            setShowEditTaskModal(null);
        } catch (e) { console.error("Error editing task:", e); }
    }

    // 4. Complete/Edit Task Status (Handles Strikethrough or Deletion upon completion)
    const handleCompleteTask = async (subjectId, taskId, action) => {
        if (!userId || !db) return;
        const subjectDocRef = doc(db, `artifacts/${appId}/users/${userId}/subjects`, subjectId);
        const subjectData = subjects.find(s => s.id === subjectId);
        const taskToUpdate = subjectData.tasks.find(t => t.id === taskId);
        if (!taskToUpdate) return;

        // Issue credit if the task wasn't already completed and has credit value
        if (taskToUpdate.credits > 0 && !taskToUpdate.isCompleted) {
            const newBalance = credit.balance + taskToUpdate.credits;
            await updateDoc(getCreditDocRef(userId), {
                balance: newBalance,
                history: arrayUnion({
                    id: crypto.randomUUID(), // Unique ID for history item
                    type: 'Credit Issued',
                    amount: taskToUpdate.credits,
                    date: new Date().toISOString(),
                    note: `Completed task: ${taskToUpdate.text}`,
                    taskId: taskId
                })
            });
        }

        // Apply deletion or strikethrough based on action
        const updatedTasks = subjectData.tasks.map(t => {
            if (t.id === taskId) {
                if (action === 'strikethrough') {
                    return { ...t, isCompleted: true, isStrikethrough: true };
                } else if (action === 'delete') {
                    return null; // Flag for permanent deletion
                }
            }
            return t;
        }).filter(t => t !== null); // Remove deleted task

        try {
            await updateDoc(subjectDocRef, { tasks: updatedTasks });
        } catch (e) { console.error("Error updating task status:", e); }
    };
    
    // 5. Delete Task Permanently (for incomplete tasks)
    const handleDeleteTaskPermanent = async (subjectId, taskId) => {
        if (!userId || !db) return;
        const subjectDocRef = doc(db, `artifacts/${appId}/users/${userId}/subjects`, subjectId);
        const subjectData = subjects.find(s => s.id === subjectId);
        const taskToUpdate = subjectData.tasks.find(t => t.id === taskId);
        if (!taskToUpdate) return;

        const updatedTasks = subjectData.tasks.filter(t => t.id !== taskId);

        try {
            await updateDoc(subjectDocRef, { tasks: updatedTasks });
        } catch (e) { console.error("Error deleting task:", e); }
    };


    // 6. Delete Subject (Main Folder)
    const handleDeleteSubject = async (subjectId) => {
        if (!userId || !db) return;
        const subjectDocRef = doc(db, `artifacts/${appId}/users/${userId}/subjects`, subjectId);
        try {
            // Note: Deleting the subject document automatically deletes all sub-folders and tasks stored within its data structure.
            await deleteDoc(subjectDocRef);
            // Optionally reset current subject ID if the deleted subject was selected in the timer
            if (currentSubjectId === subjectId) {
                setCurrentSubjectId(null);
            }
        } catch (e) { console.error("Error deleting subject:", e); }
    };
    
    // 7. Delete Sub-Folder and its tasks
    const handleDeleteFolder = async (subjectId, folderId) => {
        if (!userId || !db || !folderId) return;
        const subjectDocRef = doc(db, `artifacts/${appId}/users/${userId}/subjects`, subjectId);
        const subjectData = subjects.find(s => s.id === subjectId);
        if (!subjectData) return;

        const updatedFolders = subjectData.folders.filter(f => f.id !== folderId);
        const updatedTasks = subjectData.tasks.filter(t => t.folderId !== folderId); // Remove all tasks in that folder

        try {
            await updateDoc(subjectDocRef, {
                folders: updatedFolders,
                tasks: updatedTasks,
            });
        } catch (e) { console.error("Error deleting folder:", e); }
    };

    // 8. Add Grade
    const handleAddGrade = async (grade) => {
        if (!userId || !db) return;
        try {
            // Using setDoc with a new random ID to add a document
            await setDoc(doc(getGradesCollectionRef(userId)), {
                ...grade,
                marksObtained: parseInt(grade.marksObtained),
                createdAt: new Date().toISOString(),
            });
        } catch (e) { console.error("Error adding grade:", e); }
    };
    
    // 9. Delete Grade
    const handleDeleteGrade = async (gradeId) => {
        if (!userId || !db) return;
        const gradeDocRef = doc(db, `artifacts/${appId}/users/${userId}/grades`, gradeId);
        try {
            await deleteDoc(gradeDocRef);
        } catch (e) {
            console.error("Error deleting grade:", e);
        }
    };

    // 10. Delete Credit History Item
    const handleDeleteCreditHistory = async (historyId) => {
        if (!userId || !db) return;
        const creditDocRef = getCreditDocRef(userId);
        
        // Filter the current local history array to remove the item
        const updatedHistory = credit.history.filter(item => item.id !== historyId);

        try {
            // Overwrite the history array directly to ensure consistency across the listener
            await updateDoc(creditDocRef, { history: updatedHistory });
        } catch (e) {
            console.error("Error deleting credit history item:", e);
        }
    };
    
    // 11. Delete All Time Logs for a Selected Date
    const handleDeleteDayLog = async (date) => {
        if (!userId || !db || !date) return;
        
        // Find all time log documents for the selected date
        const timeLogCollection = getTimeLogCollectionRef(userId);

        try {
            // FIX: Use writeBatch(db) instead of db.batch()
            const batch = writeBatch(db); 
            
            // Firestore queries are needed to find docs by date, but since we cannot use 'where' here, 
            // we fetch all and filter in memory, then find the corresponding doc IDs to delete.
            const snapshot = await getDocs(timeLogCollection);
            
            let deletionCount = 0;
            snapshot.forEach(doc => {
                if (doc.data().date === date) {
                    batch.delete(doc.ref);
                    deletionCount++;
                }
            });
            
            if (deletionCount > 0) {
                await batch.commit();
            }
            setShowDeleteLogModal(false);
        } catch (e) {
            console.error("Error deleting day logs:", e);
            setErrorMessage(`Failed to delete logs for ${date}.`);
        }
    };


    // --- Timer and Reminder Logic ---
    const [timerState, setTimerState] = useState({
        isRunning: false,
        elapsedSeconds: 0,
        currentSubjectId: null,
        reminderCounter: 0,
        lastReminderTime: Date.now(),
        startTime: null,
        startLogDate: null, // New field to store YYYY-MM-DD when timer starts
        reminderIntervalSeconds: 3600, // Default 1 hour
        reminderText: "Time for a break! Drink water and stretch." // Default message
    });
    const intervalRef = useRef(null);
    

    const showNotification = (message) => {
        setNotification(message);
        setTimeout(() => setNotification(null), 5000);
    };
    
    // playReminderSound calls the Web Audio API
    const playReminderSoundSafe = () => {
        if (isAudioInitialized) {
            generateSound(soundStyle); 
        }
    };


    const handleTimerToggle = async (subjectId, intervalMinutes = 60, reminderText = "Time for a break! Drink water and stretch.") => {
        if (!userId) return;

        if (timerState.isRunning) {
            // STOP TIMER LOGIC
            clearInterval(intervalRef.current);
            intervalRef.current = null;
            setTimerState(s => ({ ...s, isRunning: false }));

            if (timerState.elapsedSeconds > 0 && timerState.startLogDate) {
                const minutesSpent = Math.ceil(timerState.elapsedSeconds / 60);
                if (minutesSpent === 0) return;

                const logDate = timerState.startLogDate; // Use the fixed start date
                const logDocRef = doc(db, `artifacts/${appId}/users/${userId}/time_logs`, `${logDate}_${timerState.currentSubjectId}`);

                try {
                    let existingDuration = 0;
                    const existingLog = timeLogs.find(log => log.date === logDate && log.subjectId === timerState.currentSubjectId);
                    if(existingLog) {
                        existingDuration = existingLog.durationMinutes || 0;
                    }
                    
                    await setDoc(logDocRef, {
                        date: logDate,
                        subjectId: timerState.currentSubjectId,
                        durationMinutes: existingDuration + minutesSpent,
                    }, { merge: true });

                } catch (e) { console.error("Error logging time:", e); }

                // Reset state after logging
                setTimerState(s => ({
                    ...s,
                    isRunning: false,
                    elapsedSeconds: 0,
                    currentSubjectId: null,
                    reminderCounter: 0,
                    lastReminderTime: Date.now(),
                    startTime: null,
                    startLogDate: null,
                }));
            }
        } else {
            // START TIMER LOGIC
            if (!subjectId) {
                setErrorMessage("Please select a subject before starting the timer.");
                return;
            }
            setErrorMessage('');
            
            // Check if audio is initialized and alert if not
            if (!isAudioInitialized) {
                setErrorMessage("Please click 'Click to Enable Sound' before starting the timer.");
                return;
            }

            const intervalSeconds = intervalMinutes * 60;
            const now = new Date();

            setTimerState(s => ({
                ...s,
                isRunning: true,
                elapsedSeconds: 0,
                currentSubjectId: subjectId,
                reminderCounter: 0,
                lastReminderTime: Date.now(),
                startTime: now.toISOString(), 
                startLogDate: formatDate(now), // Store the start date immediately
                reminderIntervalSeconds: intervalSeconds, 
                reminderText: reminderText, 
            }));

            intervalRef.current = setInterval(() => {
                setTimerState(s => {
                    const newElapsed = s.elapsedSeconds + 1;
                    const newCounter = s.reminderCounter + 1;

                    if (s.reminderIntervalSeconds > 0 && newCounter >= s.reminderIntervalSeconds) {
                        playReminderSoundSafe(); 
                        showNotification(s.reminderText); 
                        return { ...s, elapsedSeconds: newElapsed, reminderCounter: 0, lastReminderTime: Date.now() };
                    }

                    return { ...s, elapsedSeconds: newElapsed, reminderCounter: newCounter };
                });
            }, 1000);
        }
    };

    const formatTime = (totalSeconds) => {
        const h = Math.floor(totalSeconds / 3600);
        const m = Math.floor((totalSeconds % 3600) / 60);
        const s = totalSeconds % 60;
        return `${h.toString().padStart(2, '0')}:${m.toString().padStart(2, '0')}:${s.toString().padStart(2, '0')}`;
    };

    // --- Data Calculation for Calendar View ---
    const calculateDailyTime = useMemo(() => {
        const dailyTotals = {};
        timeLogs.forEach(log => {
            const { date, subjectId, durationMinutes } = log;
            if (!dailyTotals[date]) {
                dailyTotals[date] = { total: 0, subjects: {} };
            }
            dailyTotals[date].total += durationMinutes;
            dailyTotals[date].subjects[subjectId] = (dailyTotals[date].subjects[subjectId] || 0) + durationMinutes;
        });
        return dailyTotals;
    }, [timeLogs]);
    
    // --- Task Priority/Coloring Logic ---
    const getTaskPriority = (task) => {
        if (task.isCompleted) return { priority: 5, color: 'gray', accent: 'gray' };
        if (task.color) return { priority: 0, color: task.color, accent: task.color }; // User choice overrides auto-priority
        if (!task.deadline) return { priority: 4, color: 'gray', accent: 'gray' };

        const deadlineTime = new Date(task.deadline).getTime();
        const now = Date.now();
        const diffHours = (deadlineTime - now) / (1000 * 60 * 60);

        if (diffHours <= 0) return { priority: 1, color: 'red', accent: 'red' }; // Overdue
        if (diffHours <= 48) return { priority: 2, color: 'orange', accent: 'orange' }; // Urgent (<= 2 days)
        if (diffHours <= 7 * 24) return { priority: 3, color: 'yellow', accent: 'yellow' }; // Medium (<= 7 days)
        
        return { priority: 4, color: 'blue', accent: 'blue' }; // Low (> 7 days)
    };

    const sortTasks = (tasks) => {
        return tasks.sort((a, b) => {
            const priorityA = getTaskPriority(a);
            const priorityB = getTaskPriority(b);

            // Completed tasks and tasks without deadlines go to the bottom
            if (priorityA.priority === 5 && priorityB.priority !== 5) return 1;
            if (priorityA.priority !== 5 && priorityB.priority === 5) return -1;
            
            // Sort by priority (1=Highest, 4=Lowest)
            if (priorityA.priority !== priorityB.priority) {
                return priorityA.priority - priorityB.priority;
            }

            // Secondary sort: closest deadline first
            const deadlineA = a.deadline ? new Date(a.deadline).getTime() : Infinity;
            const deadlineB = b.deadline ? new Date(b.deadline).getTime() : Infinity;
            return deadlineA - deadlineB;
        });
    };

    // --- Helper Components ---
    
    const ConfirmationModal = ({ isOpen, onClose, onConfirm, title, message, confirmText = 'Confirm Deletion', confirmColor = 'bg-red-600' }) => {
        if (!isOpen) return null;
        return (
            <div className="fixed inset-0 bg-black bg-opacity-50 z-50 flex items-center justify-center p-4">
                <div className="bg-white rounded-xl shadow-2xl w-full max-w-sm transition-all duration-300 transform scale-100">
                    <div className="p-6 space-y-4">
                        <h3 className="text-xl font-bold text-gray-800 flex items-center">
                            <AlertTriangle size={24} className="text-red-500 mr-2"/> {title}
                        </h3>
                        <p className="text-gray-600 text-sm">{message}</p>
                        <div className="flex gap-3 justify-end">
                            <button onClick={onClose} className="px-4 py-2 bg-gray-200 text-gray-700 font-semibold rounded-lg hover:bg-gray-300 transition-colors">
                                Cancel
                            </button>
                            <button onClick={onConfirm} className={`px-4 py-2 text-white font-semibold rounded-lg transition-colors ${confirmColor} hover:opacity-90`}>
                                {confirmText}
                            </button>
                        </div>
                    </div>
                </div>
            </div>
        );
    };

    const IconWrapper = ({ children, onClick, title, active }) => (
        <button
            onClick={onClick}
            title={title}
            className={`p-3 rounded-xl transition-all duration-300 ${active ? 'bg-indigo-600 text-white shadow-lg shadow-indigo-500/50' : 'bg-gray-100 text-gray-600 hover:bg-indigo-50 hover:text-indigo-600'} flex flex-col items-center justify-center`}
        >
            {children}
        </button>
    );

    const Modal = ({ isOpen, onClose, title, children }) => {
        if (!isOpen) return null;
        return (
            <div className="fixed inset-0 bg-black bg-opacity-50 z-50 flex items-center justify-center p-4">
                <div className="bg-white rounded-xl shadow-2xl w-full max-w-lg transition-all duration-300 transform scale-100">
                    <div className="flex justify-between items-center p-4 border-b">
                        <h3 className="text-xl font-bold text-gray-800">{title}</h3>
                        <button onClick={onClose} className="p-2 rounded-full hover:bg-gray-100 text-gray-500">
                            <X size={20} />
                        </button>
                    </div>
                    <div className="p-4">{children}</div>
                </div>
            </div>
        );
    };

    const NotificationToast = ({ message, onClose }) => {
        if (!message) return null;
        return (
            <div className="fixed bottom-4 right-4 z-50 p-4 bg-indigo-600 text-white rounded-xl shadow-2xl flex items-center space-x-3 transition-transform duration-300 transform animate-slide-in">
                <Bell size={24} />
                <p className="font-semibold">{message}</p>
                <button onClick={onClose} className="text-white opacity-70 hover:opacity-100">
                    <X size={16} />
                </button>
            </div>
        );
    };
    
    // --- Task/Folder Forms (Moved outside FolderView) ---

    const SubjectForm = ({ onSubmit }) => {
        const [name, setName] = useState('');
        const handleSubmit = (e) => {
            e.preventDefault();
            if (name) {
                onSubmit(name);
                setName('');
            }
        };
        return (
            <form onSubmit={handleSubmit} className="flex gap-2">
                <input
                    type="text"
                    value={name}
                    onChange={(e) => setName(e.target.value)}
                    placeholder="Subject Name (e.g., Math)"
                    className="p-2 border border-gray-300 rounded-lg flex-grow focus:ring-indigo-500 focus:border-indigo-500"
                    required
                />
                <button type="submit" className="px-4 py-2 bg-indigo-500 text-white rounded-lg hover:bg-indigo-600 transition-colors">
                    Add
                </button>
            </form>
        );
    };

    const SubjectEditForm = ({ subject, onSave, onClose }) => {
        const [name, setName] = useState(subject.name);
        const handleSubmit = (e) => {
            e.preventDefault();
            if (name) {
                onSave(subject.id, name);
            }
        };
        return (
            <form onSubmit={handleSubmit} className="space-y-4">
                <div>
                    <label className="block text-sm font-medium text-gray-700">Subject Name</label>
                    <input type="text" value={name} onChange={(e) => setName(e.target.value)} className="w-full p-2 border rounded-lg" required />
                </div>
                <button type="submit" className="w-full py-2 bg-indigo-500 text-white font-semibold rounded-lg hover:bg-indigo-600 transition-colors">
                    Save Changes
                </button>
            </form>
        );
    };

    const FolderForm = ({ subjectId, initialFolder, onSave, onClose }) => {
        const isEditing = !!initialFolder;
        const [name, setName] = useState(initialFolder?.name || '');
        const [description, setDescription] = useState(initialFolder?.description || '');
        const [color, setColor] = useState(initialFolder?.color || '');

        const handleSubmit = (e) => {
            e.preventDefault();
            if (name) {
                const updatedFolder = {
                    id: initialFolder?.id || crypto.randomUUID(),
                    name,
                    description,
                    color,
                };
                if (isEditing) {
                    onSave(subjectId, updatedFolder);
                } else {
                    onSave(subjectId, { name, description, color });
                }
                onClose();
            }
        };

        return (
            <form onSubmit={handleSubmit} className="space-y-4">
                <div>
                    <label className="block text-sm font-medium text-gray-700">Folder Name</label>
                    <input type="text" value={name} onChange={(e) => setName(e.target.value)} placeholder="e.g., Chapter 1 Notes" className="w-full p-2 border rounded-lg" required />
                </div>
                <div>
                    <label className="block text-sm font-medium text-gray-700">Description (Optional)</label>
                    <textarea value={description} onChange={(e) => setDescription(e.target.value)} placeholder="Details for this folder..." rows="2" className="w-full p-2 border rounded-lg"></textarea>
                </div>
                <div>
                    <label className="block text-sm font-medium text-gray-700">Custom Color Highlight (Optional)</label>
                    <select value={color} onChange={(e) => setColor(e.target.value)} className="w-full p-2 border rounded-lg">
                        {COLOR_OPTIONS.map(opt => (
                            <option key={opt.value} value={opt.value}>{opt.name}</option>
                        ))}
                    </select>
                </div>
                <button type="submit" className="w-full py-2 bg-indigo-500 text-white font-semibold rounded-lg hover:bg-indigo-600 transition-colors">
                    {isEditing ? 'Save Changes' : 'Create Folder'}
                </button>
            </form>
        );
    };
    
    const TaskForm = ({ onSubmit, subjectId, folderId, subjectMap, initialTask, onClose }) => {
        const isEditing = !!initialTask;
        const subjectName = subjectMap[subjectId]?.name || 'Unknown';
        const folderName = folderId ? subjectMap[subjectId]?.folders?.find(f => f.id === folderId)?.name : 'Main Subject';

        const [text, setText] = useState(initialTask?.text || '');
        const [description, setDescription] = useState(initialTask?.description || '');
        const [deadline, setDeadline] = useState(initialTask?.deadline || '');
        const [credits, setCredits] = useState(initialTask?.credits || '');
        const [color, setColor] = useState(initialTask?.color || '');


        const handleSubmit = (e) => {
            e.preventDefault();
            if (text && credits !== '') {
                const taskData = { 
                    id: initialTask?.id || crypto.randomUUID(),
                    folderId: folderId || null, 
                    text, 
                    description,
                    deadline, 
                    credits: parseFloat(credits),
                    color,
                    isCompleted: initialTask?.isCompleted || false,
                    isStrikethrough: initialTask?.isStrikethrough || false,
                    createdAt: initialTask?.createdAt || new Date().toISOString()
                };

                onSubmit(subjectId, taskData);
                onClose(); // Close on successful submission
            }
        };

        return (
            <form onSubmit={handleSubmit} className="space-y-4">
                <p className="text-sm text-gray-500">
                    {isEditing ? 'Editing Task in: ' : 'Creating task in: '}
                    <span className="font-semibold text-indigo-600">{subjectName} / {folderName}</span>
                </p>
                <div>
                    <label className="block text-sm font-medium text-gray-700">Task Title</label>
                    <input type="text" value={text} onChange={(e) => setText(e.target.value)} placeholder="E.g., Finish Chapter 3 homework" className="w-full p-2 border rounded-lg" required />
                </div>
                <div>
                    <label className="block text-sm font-medium text-gray-700">Description (Optional)</label>
                    <textarea value={description} onChange={(e) => setDescription(e.target.value)} placeholder="Detailed steps or notes..." rows="2" className="w-full p-2 border rounded-lg"></textarea>
                </div>
                <div>
                    <label className="block text-sm font-medium text-gray-700">Deadline (Date & Time)</label>
                    <input type="datetime-local" value={deadline} onChange={(e) => setDeadline(e.target.value)} className="w-full p-2 border rounded-lg" />
                </div>
                <div>
                    <label className="block text-sm font-medium text-gray-700">Credits for Completion ($)</label>
                    <input type="number" value={credits} onChange={(e) => setCredits(e.target.value)} placeholder="0.00" step="0.01" className="w-full p-2 border rounded-lg" required />
                </div>
                <div>
                    <label className="block text-sm font-medium text-gray-700">Custom Color Highlight (Optional)</label>
                    <select value={color} onChange={(e) => setColor(e.target.value)} className="w-full p-2 border rounded-lg">
                        {COLOR_OPTIONS.map(opt => (
                            <option key={opt.value} value={opt.value}>{opt.name}</option>
                        ))}
                    </select>
                </div>
                <button type="submit" className="w-full py-2 bg-green-500 text-white font-semibold rounded-lg hover:bg-green-600 transition-colors">
                    {isEditing ? 'Save Task Changes' : 'Create Task'}
                </button>
            </form>
        );
    };

    const FolderView = () => {
        const [showAddSubject, setShowAddSubject] = useState(false);
        const [showAddFolder, setShowAddFolder] = useState(null); // subjectId
        const [showAddTask, setShowAddTask] = useState(null); // {subjectId, folderId}
        const [showCompleteModal, setShowCompleteModal] = useState(null); // {subjectId, taskId}

        const subjectMap = useMemo(() => subjects.reduce((acc, sub) => ({ ...acc, [sub.id]: sub }), {}), [subjects]);
        
        // Map priority colors to Tailwind classes
        const getColorClasses = (priority, color) => {
            if (color) {
                return {
                    border: `border-${color}-500`,
                    bg: `bg-${color}-50`,
                    text: `text-${color}-700`,
                    accent: `text-${color}-500`
                }
            }
            switch (priority) {
                case 1: return { border: 'border-red-500', bg: 'bg-red-50', text: 'text-red-700', accent: 'text-red-500' }; // Overdue
                case 2: return { border: 'border-orange-500', bg: 'bg-orange-50', text: 'text-orange-700', accent: 'text-orange-500' }; // Urgent
                case 3: return { border: 'border-yellow-500', bg: 'bg-yellow-50', text: 'text-yellow-700', accent: 'text-yellow-500' }; // Medium
                case 4: return { border: 'border-blue-500', bg: 'bg-blue-50', text: 'text-blue-700', accent: 'text-blue-500' }; // Low / No Deadline
                case 5: return { border: 'border-gray-300', bg: 'bg-gray-100', text: 'text-gray-500', accent: 'text-gray-500' }; // Completed
                default: return { border: 'border-gray-200', bg: 'bg-white', text: 'text-gray-800', accent: 'text-gray-600' };
            }
        };


        const TaskItem = ({ task, subjectId }) => {
            const priorityInfo = getTaskPriority(task);
            const classes = getColorClasses(priorityInfo.priority, task.color);

            return (
                <div className={`p-3 border-l-4 ${classes.border} rounded-lg flex items-start justify-between transition-colors ${classes.bg}`}>
                    <div className="flex-grow">
                        <p className={`font-medium ${task.isStrikethrough ? 'line-through text-gray-500' : classes.text}`}>
                            {task.text}
                        </p>
                        {task.description && (
                            <p className="text-xs text-gray-600 mt-1 italic">{task.description}</p>
                        )}
                        {task.deadline && (
                            <p className="text-xs text-indigo-500 mt-1">Deadline: {new Date(task.deadline).toLocaleString()}</p>
                        )}
                        <p className="text-sm text-amber-600 mt-1 font-semibold">${task.credits.toFixed(2)} Credit</p>
                    </div>
                    <div className="flex space-x-2 ml-4 flex-shrink-0">
                        {/* Edit Button */}
                        <button
                            onClick={() => setShowEditTaskModal({ subjectId, task })}
                            className="p-1 bg-indigo-100 text-indigo-600 rounded-full hover:bg-indigo-200"
                            title="Edit Task"
                        >
                            <Edit2 size={18} />
                        </button>
                        
                        {!task.isCompleted ? (
                            <>
                                {/* Delete incomplete task permanently */}
                                <button
                                    onClick={() => handleDeleteTaskPermanent(subjectId, task.id)}
                                    className="p-1 bg-red-100 text-red-600 rounded-full hover:bg-red-200"
                                    title="Delete Task Permanarily"
                                >
                                    <Trash2 size={18} />
                                </button>
                                <button
                                    onClick={() => setShowCompleteModal({ subjectId, taskId: task.id })}
                                    className="p-1 bg-green-500 text-white rounded-full hover:bg-green-600"
                                    title="Mark as Complete"
                                >
                                    <CheckCircle size={18} />
                                </button>
                            </>
                        ) : (
                            // If already completed (strikethrough), the only action is permanent deletion
                            <button
                                onClick={() => handleDeleteTaskPermanent(subjectId, task.id)}
                                className="p-1 bg-red-500 text-white rounded-full hover:bg-red-600"
                                title="Delete Task"
                            >
                                <Trash2 size={18} />
                            </button>
                        )}
                    </div>
                </div>
            );
        };
        
        // Reusable Task and Folder Modals
        const TaskAddModal = (
             <Modal isOpen={!!showAddTask} onClose={() => setShowAddTask(null)} title="Create New Task">
                <TaskForm 
                    // FIX: Pass all arguments correctly to the handler function signature (subjectId, folderId, taskData)
                    onSubmit={(subjectId, taskData) => handleAddTask(subjectId, showAddTask.folderId, taskData)} 
                    subjectId={showAddTask?.subjectId} 
                    folderId={showAddTask?.folderId} 
                    subjectMap={subjectMap} 
                    onClose={() => setShowAddTask(null)}
                />
            </Modal>
        );

        const TaskEditModal = showEditTaskModal && (
            <Modal isOpen={true} onClose={() => setShowEditTaskModal(null)} title="Edit Task">
                <TaskForm 
                    onSubmit={handleEditTask} 
                    subjectId={showEditTaskModal.subjectId} 
                    folderId={showEditTaskModal.task.folderId} 
                    subjectMap={subjectMap} 
                    initialTask={showEditTaskModal.task}
                    onClose={() => setShowEditTaskModal(null)}
                />
            </Modal>
        );
        
        const FolderAddModal = (
            <Modal isOpen={!!showAddFolder} onClose={() => setShowAddFolder(null)} title={`Create Folder in ${subjectMap[showAddFolder]?.name || ''}`}>
                <FolderForm 
                    subjectId={showAddFolder} 
                    onSave={handleAddFolder} 
                    onClose={() => setShowAddFolder(null)}
                />
            </Modal>
        );

        const FolderEditModal = showEditFolderModal && (
            <Modal isOpen={true} onClose={() => setShowEditFolderModal(null)} title={`Edit Folder: ${showEditFolderModal.folder.name}`}>
                <FolderForm 
                    subjectId={showEditFolderModal.subjectId} 
                    initialFolder={showEditFolderModal.folder} 
                    onSave={handleEditFolder} 
                    onClose={() => setShowEditFolderModal(null)}
                />
            </Modal>
        );


        return (
            <div className="p-4 space-y-4">
                <h2 className="text-2xl font-bold text-gray-800">Task Management</h2>

                <button
                    onClick={() => setShowAddSubject(true)}
                    className="w-full py-3 bg-indigo-600 text-white font-semibold rounded-xl shadow-md hover:bg-indigo-700 transition-colors"
                >
                    + New Subject (Main Folder)
                </button>

                <Modal isOpen={showAddSubject} onClose={() => setShowAddSubject(false)} title="Create New Subject">
                    <SubjectForm onSubmit={(name) => { handleAddSubject(name); setShowAddSubject(false); }} />
                </Modal>
                {FolderAddModal}
                {TaskAddModal}
                {TaskEditModal}
                {FolderEditModal}
                <Modal isOpen={!!showEditSubjectModal} onClose={() => setShowEditSubjectModal(null)} title={`Edit Subject: ${showEditSubjectModal?.name || ''}`}>
                    <SubjectEditForm 
                        subject={showEditSubjectModal} 
                        onSave={handleEditSubject} 
                        onClose={() => setShowEditSubjectModal(null)} 
                    />
                </Modal>
                
                <Modal isOpen={!!showCompleteModal} onClose={() => setShowCompleteModal(null)} title="Task Completion">
                    <div className="p-4 space-y-4">
                        <p className="text-lg font-medium text-gray-700">
                            Task completed! **Credit will be issued** regardless of the choice below.
                        </p>
                        <p className="text-sm text-red-500"><AlertTriangle size={16} className="inline-block mr-1"/> Choose carefully: Strikethrough keeps the log; Deletion removes it permanently.</p>
                        <div className="flex gap-4">
                            <button
                                onClick={() => { handleCompleteTask(showCompleteModal.subjectId, showCompleteModal.taskId, 'strikethrough'); setShowCompleteModal(null); }}
                                className="flex-1 py-3 bg-yellow-500 text-white font-semibold rounded-lg hover:bg-yellow-600 transition-colors"
                            >
                                Strikethrough
                            </button>
                            <button
                                onClick={() => { handleCompleteTask(showCompleteModal.subjectId, showCompleteModal.taskId, 'delete'); setShowCompleteModal(null); }}
                                className="flex-1 py-3 bg-red-500 text-white font-semibold rounded-lg hover:bg-red-600 transition-colors"
                            >
                                Complete Deletion
                            </button>
                        </div>
                    </div>
                </Modal>


                <div className="space-y-6">
                    {subjects.length === 0 ? (
                        <p className="text-center text-gray-500 mt-8">No subjects created yet. Click the button above!</p>
                    ) : (
                        subjects.map(subject => (
                            <div key={subject.id} className="border border-gray-300 rounded-xl p-4 shadow-lg bg-white">
                                <div className="flex items-center justify-between mb-3 border-b pb-2">
                                    <h3 className="text-xl font-extrabold text-indigo-700 flex items-center">
                                        <Folder size={20} className="mr-2"/> {subject.name}
                                    </h3>
                                    <div className="flex gap-2">
                                        <button 
                                            onClick={() => setShowEditSubjectModal(subject)} 
                                            className="text-sm text-gray-600 hover:text-indigo-600 p-1 rounded-md border"
                                            title="Edit Subject Name"
                                        >
                                            <Edit2 size={16}/>
                                        </button>
                                        <button onClick={() => setShowAddFolder(subject.id)} className="text-sm text-gray-600 hover:text-indigo-600 p-1 rounded-md border">
                                            + Folder
                                        </button>
                                        <button onClick={() => setShowAddTask({ subjectId: subject.id, folderId: null })} className="text-sm text-white bg-green-500 hover:bg-green-600 p-1 px-2 rounded-md">
                                            + Task
                                        </button>
                                        {/* Delete Subject Button */}
                                        <button 
                                            onClick={() => handleDeleteSubject(subject.id)} 
                                            className="text-sm text-white bg-red-500 hover:bg-red-600 p-1 px-2 rounded-md flex items-center"
                                            title="Delete Subject and ALL Contents"
                                        >
                                            <Trash2 size={14} className="mr-1"/>Delete
                                        </button>
                                    </div>
                                </div>

                                <div className="space-y-3">
                                    {/* Tasks not assigned to a folder */}
                                    {sortTasks(subject.tasks.filter(t => !t.folderId)).map(task => (
                                        <TaskItem key={task.id} task={task} subjectId={subject.id} />
                                    ))}

                                    {/* Sub-Folders and their Tasks */}
                                    {subject.folders.map(folder => {
                                        const folderTasks = sortTasks(subject.tasks.filter(t => t.folderId === folder.id));
                                        const folderClasses = getColorClasses(0, folder.color); // Folder uses only custom color

                                        return (
                                            <div key={folder.id} className="ml-4 border-l-4 border-gray-200 pl-3">
                                                <div className={`flex items-center justify-between p-2 rounded-t-lg transition-colors ${folderClasses.bg}`}>
                                                    <div className="flex-1">
                                                        <h4 className={`font-semibold ${folderClasses.text} flex items-center`}>
                                                            <List size={16} className={`mr-1 ${folderClasses.accent}`}/> {folder.name}
                                                        </h4>
                                                        {folder.description && (
                                                             <p className="text-xs text-gray-600 italic">{folder.description}</p>
                                                        )}
                                                    </div>
                                                    
                                                    <div className="flex gap-2 flex-shrink-0">
                                                        <button 
                                                            onClick={() => setShowEditFolderModal({ subjectId: subject.id, folder })} 
                                                            className="text-xs text-gray-600 hover:text-indigo-600 p-1 rounded-md"
                                                            title="Edit Folder"
                                                        >
                                                            <Edit2 size={14}/>
                                                        </button>
                                                        <button onClick={() => setShowAddTask({ subjectId: subject.id, folderId: folder.id })} className="text-xs text-indigo-600 hover:text-indigo-800">
                                                            + Task
                                                        </button>
                                                        {/* Delete Folder Button */}
                                                        <button 
                                                            onClick={() => handleDeleteFolder(subject.id, folder.id)} 
                                                            className="text-xs text-red-600 hover:text-red-800"
                                                            title="Delete Folder and ALL contained tasks"
                                                        >
                                                            <Trash2 size={14} className="inline-block"/>
                                                        </button>
                                                    </div>
                                                </div>
                                                <div className="space-y-2 pt-2">
                                                    {folderTasks.map(task => (
                                                        <TaskItem key={task.id} task={task} subjectId={subject.id} />
                                                    ))}
                                                </div>
                                            </div>
                                        );
                                    })}
                                </div>
                            </div>
                        ))
                    )}
                </div>
            </div>
        );
    };

    const TimerView = () => {
        const currentSubject = subjects.find(s => s.id === timerState.currentSubjectId);
        const [showReminderConfig, setShowReminderConfig] = useState(false);
        const [reminderInterval, setReminderInterval] = useState(60); // In minutes
        const [reminderMessage, setReminderMessage] = useState("Time for a break! Drink water and stretch.");
        

        const handleStartClick = () => {
            if (!currentSubjectId) {
                setErrorMessage("Please select a subject before starting the timer.");
                return;
            }
            setShowReminderConfig(true);
        };

        const handleConfigAndStart = () => {
            if (reminderInterval < 1) {
                setReminderInterval(60); // Default to 60 if invalid input
            }
            
            // Check if audio is initialized and alert if not
            if (!isAudioInitialized) {
                setErrorMessage("Please click 'Click to Enable Sound' before starting the timer.");
                return;
            }

            handleTimerToggle(currentSubjectId, reminderInterval, reminderMessage);
            setShowReminderConfig(false);
        };
        
        const handleStopClick = () => {
            // Call handleTimerToggle with existing state (no need for arguments, stop logic handles it)
            handleTimerToggle(currentSubjectId); 
        };

        return (
            <div className="p-4 text-center space-y-8">
                <h2 className="text-3xl font-extrabold text-indigo-700">Study Time Tracker</h2>
                <div className="p-8 bg-white rounded-3xl shadow-2xl space-y-6">
                    <p className="text-5xl font-mono font-bold text-gray-900">{formatTime(timerState.elapsedSeconds)}</p>

                    {timerState.isRunning && currentSubject ? (
                        <p className="text-lg text-green-600 font-semibold">Working on: <span className="font-extrabold">{currentSubject.name}</span></p>
                    ) : (
                        <p className="text-lg text-gray-500 font-semibold">
                            {currentSubjectId ? `Ready for: ${subjects.find(s => s.id === currentSubjectId)?.name}` : 'Select a subject to begin.'}
                        </p>
                    )}

                    <div className="flex justify-center">
                        <button
                            onClick={timerState.isRunning ? handleStopClick : handleStartClick}
                            className={`px-8 py-4 text-2xl font-bold rounded-full transition-all duration-300 shadow-xl ${
                                timerState.isRunning
                                    ? 'bg-red-600 hover:bg-red-700 text-white shadow-red-500/50'
                                    : 'bg-green-600 hover:bg-green-700 text-white shadow-green-500/50'
                            }`}
                            disabled={!currentSubjectId && !timerState.isRunning}
                        >
                            {timerState.isRunning ? 'STOP' : 'START'}
                        </button>
                    </div>

                    {timerState.isRunning && (
                        <div className="mt-4 p-3 bg-yellow-50 rounded-lg text-yellow-800 font-medium border border-yellow-300 flex items-center justify-center">
                            <Clock size={18} className="inline-block mr-2"/> Reminder: "{timerState.reminderText}" every {timerState.reminderIntervalSeconds / 60} minutes.
                        </div>
                    )}
                </div>

                <div className="space-y-4">
                    <h3 className="text-xl font-bold text-gray-800">Select Subject</h3>
                    <div className="grid grid-cols-2 sm:grid-cols-3 gap-3">
                        {subjects.map(subject => (
                            <button
                                key={subject.id}
                                onClick={() => setCurrentSubjectId(subject.id)}
                                disabled={timerState.isRunning && timerState.currentSubjectId !== subject.id}
                                className={`p-3 rounded-xl border-2 font-semibold transition-all ${
                                    currentSubjectId === subject.id
                                        ? 'border-indigo-600 bg-indigo-50 text-indigo-700'
                                        : 'border-gray-200 hover:bg-gray-50 text-gray-600'
                                } ${timerState.isRunning && timerState.currentSubjectId !== subject.id ? 'opacity-50 cursor-not-allowed' : ''}`}
                            >
                                {subject.name}
                            </button>
                        ))}
                    </div>
                </div>
                
                {/* Custom Reminder Configuration Modal */}
                <Modal isOpen={showReminderConfig} onClose={() => setShowReminderConfig(false)} title={`Set Reminder for ${currentSubject?.name || 'Study Session'}`}>
                    <div className="space-y-4">
                        
                        {/* Sound Style Selector */}
                        <div>
                            <label className="block text-sm font-medium text-gray-700 mb-1">Sound Style</label>
                            <div className="flex gap-2">
                                <button 
                                    onClick={() => setSoundStyle('highBeep')}
                                    type="button"
                                    className={`flex-1 py-2 rounded-lg font-semibold transition-colors ${soundStyle === 'highBeep' ? 'bg-indigo-600 text-white' : 'bg-gray-200 text-gray-700 hover:bg-gray-300'}`}
                                >
                                    High Beep
                                </button>
                                <button 
                                    onClick={() => setSoundStyle('chime')}
                                    type="button"
                                    className={`flex-1 py-2 rounded-lg font-semibold transition-colors ${soundStyle === 'chime' ? 'bg-indigo-600 text-white' : 'bg-gray-200 text-gray-700 hover:bg-gray-300'}`}
                                >
                                    Chime
                                </button>
                            </div>
                        </div>

                        {/* Sound Initialization (The crucial part) */}
                        <div className="border border-indigo-200 p-3 rounded-lg bg-indigo-50">
                            <p className="font-semibold text-sm text-indigo-700 mb-2">Sound Check (REQUIRED)</p>
                            <p className="text-xs text-indigo-600 mb-3">Click the button below once to grant the browser permission to play timed reminders.</p>
                            <button 
                                onClick={testReminderSound}
                                className="w-full py-2 bg-indigo-500 text-white font-semibold rounded-lg hover:bg-indigo-600 transition-colors"
                            >
                                {isAudioInitialized ? 'Sound Enabled!' : 'Click to Enable Sound'}
                            </button>
                        </div>

                        <div>
                            <label className="block text-sm font-medium text-gray-700">Reminder Interval (Minutes)</label>
                            <input 
                                type="number" 
                                value={reminderInterval} 
                                onChange={(e) => setReminderInterval(Math.max(1, parseInt(e.target.value) || 1))}
                                placeholder="e.g., 60" 
                                className="w-full p-2 border rounded-lg" 
                                required 
                                min="1"
                            />
                        </div>
                        <div>
                            <label className="block text-sm font-medium text-gray-700">Reminder Message</label>
                            <input 
                                type="text" 
                                value={reminderMessage} 
                                onChange={(e) => setReminderMessage(e.target.value)}
                                placeholder="e.g., Stand up and stretch!" 
                                className="w-full p-2 border rounded-lg" 
                                required 
                            />
                        </div>
                        <button 
                            onClick={handleConfigAndStart}
                            className="w-full py-2 bg-green-500 text-white font-semibold rounded-lg hover:bg-green-600 transition-colors"
                        >
                            Start Timer with Custom Reminder
                        </button>
                    </div>
                </Modal>
                <NotificationToast message={notification} onClose={() => setNotification(null)} />
            </div>
        );
    };

    const CalendarView = () => {
        const [currentDate, setCurrentDate] = useState(new Date());
        const [selectedDate, setSelectedDate] = useState(formatDate(new Date()));

        const subjectMap = useMemo(() => subjects.reduce((acc, sub) => ({ ...acc, [sub.id]: sub.name }), {}), [subjects]);

        // State for Delete Day Log Confirmation Modal
        const [showDeleteModal, setShowDeleteModal] = useState(false);

        const changeMonth = (delta) => {
            setCurrentDate(prev => {
                const newDate = new Date(prev);
                newDate.setMonth(prev.getMonth() + delta);
                return newDate;
            });
        };

        const daysInMonth = (date) => new Date(date.getFullYear(), date.getMonth() + 1, 0).getDate();
        const firstDayOfMonth = (date) => {
            const day = new Date(date.getFullYear(), date.getMonth(), 1).getDay();
            // Adjusting for Monday start (0=Sun, 1=Mon, ..., 6=Sat). We want Monday to be 0 for array indexing.
            return day === 0 ? 6 : day - 1;
        }; 

        const renderCalendar = () => {
            const year = currentDate.getFullYear();
            const month = currentDate.getMonth();
            const totalDays = daysInMonth(currentDate);
            const startDay = firstDayOfMonth(currentDate); 

            const blanks = Array(startDay).fill(null); 
            const days = Array.from({ length: totalDays }, (_, i) => i + 1);

            const allDays = [...blanks, ...days];
            const weeks = [];
            for (let i = 0; i < allDays.length; i += 7) {
                weeks.push(allDays.slice(i, i + 7));
            }

            const today = formatDate(new Date());

            return (
                <div className="grid grid-cols-7 gap-1 text-xs sm:text-sm">
                    {['Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat', 'Sun'].map(day => (
                        <div key={day} className="text-center font-bold text-indigo-600 p-2 border-b-2 border-indigo-100">{day}</div>
                    ))}
                    {weeks.flat().map((day, index) => {
                        const dateKey = day ? formatDate(new Date(year, month, day)) : null;
                        const logData = dateKey ? calculateDailyTime[dateKey] : null;
                        const isSelected = dateKey === selectedDate;
                        const isToday = dateKey === today;

                        return (
                            <div
                                key={index}
                                className={`h-20 p-1 rounded-lg transition-all border cursor-pointer flex flex-col items-center relative
                                    ${day ? 'hover:bg-indigo-50' : 'bg-gray-50 text-gray-400 cursor-default'}
                                    ${isSelected ? 'bg-indigo-200 border-indigo-600 scale-105 shadow-md' : ''}
                                    ${isToday && !isSelected ? 'bg-yellow-100 border-yellow-500' : ''}
                                `}
                                onClick={() => day && setSelectedDate(dateKey)}
                            >
                                <span className={`font-semibold ${isToday ? 'text-red-600' : 'text-gray-800'}`}>{day}</span>
                                {logData && (
                                    <div className="text-center mt-1 text-xs">
                                        <p className="font-bold text-green-600">{Math.floor(logData.total / 60)}h {logData.total % 60}m</p>
                                        <p className="text-gray-500">{Object.keys(logData.subjects).length} Subjects</p>
                                    </div>
                                )}
                            </div>
                        );
                    })}
                </div>
            );
        };

        const selectedLog = calculateDailyTime[selectedDate];
        const selectedDateFormatted = new Date(selectedDate).toLocaleDateString('en-US', { weekday: 'long', year: 'numeric', month: 'long', day: 'numeric' });

        return (
            <div className="p-4 space-y-6">
                <h2 className="text-2xl font-bold text-gray-800">Study Time Calendar</h2>
                
                {/* Calendar Navigation */}
                <div className="flex justify-between items-center bg-white p-3 rounded-xl shadow-md">
                    <button onClick={() => changeMonth(-1)} className="p-2 rounded-full hover:bg-gray-100"><ChevronLeft size={24} /></button>
                    <h3 className="text-xl font-bold text-indigo-700">
                        {currentDate.toLocaleString('default', { month: 'long', year: 'numeric' })}
                    </h3>
                    <button onClick={() => changeMonth(1)} className="p-2 rounded-full hover:bg-gray-100"><ChevronRight size={24} /></button>
                </div>

                {renderCalendar()}

                {/* Daily Details View */}
                <div className="mt-8 p-4 border-t-2 border-indigo-600 bg-white rounded-xl shadow-lg">
                    <h3 className="text-xl font-bold text-gray-800 mb-3">Time Spent on {selectedDateFormatted}</h3>
                    {selectedLog ? (
                        <div className="space-y-3">
                            <p className="text-2xl font-extrabold text-indigo-600 border-b pb-2">
                                Total Study Time: {Math.floor(selectedLog.total / 60)}h {selectedLog.total % 60}m
                            </p>
                            <div className="space-y-2 pt-2">
                                {Object.entries(selectedLog.subjects).map(([subjectId, durationMinutes]) => (
                                    <div key={subjectId} className="flex justify-between items-center p-2 bg-indigo-50 rounded-lg">
                                        <span className="font-semibold text-gray-700">{subjectMap[subjectId] || 'Unknown Subject'}</span>
                                        <span className="text-indigo-800 font-medium">
                                            {Math.floor(durationMinutes / 60)}h {durationMinutes % 60}m
                                        </span>
                                    </div>
                                ))}
                            </div>
                            <button
                                onClick={() => setShowDeleteModal(true)}
                                className="w-full mt-4 py-2 bg-red-500 text-white font-semibold rounded-lg hover:bg-red-600 transition-colors flex items-center justify-center space-x-2"
                            >
                                <Trash2 size={18}/> <span>Delete Day Log</span>
                            </button>
                        </div>
                    ) : (
                        <p className="text-gray-500">No study time recorded for this date.</p>
                    )}
                </div>
                
                <ConfirmationModal
                    isOpen={showDeleteModal}
                    onClose={() => setShowDeleteModal(false)}
                    onConfirm={() => handleDeleteDayLog(selectedDate)}
                    title="Confirm Log Deletion"
                    message={`Are you sure you want to permanently delete ALL study time logs for ${selectedDateFormatted}? This cannot be undone.`}
                />
            </div>
        );
    };

    const GradesView = () => {
        const [showAddGrade, setShowAddGrade] = useState(false);
        const [newGrade, setNewGrade] = useState({ subjectId: '', type: '', date: formatDate(new Date()), marksObtained: '', totalMarks: 100 });
        
        const subjectOptions = subjects.map(s => ({ value: s.id, label: s.name }));
        
        const handleGradeChange = useCallback((e) => {
            const { name, value } = e.target;
            setNewGrade(prev => ({ ...prev, [name]: value }));
        }, []);

        const handleSubmitGrade = (e) => {
            e.preventDefault();
            if (newGrade.subjectId && newGrade.marksObtained && newGrade.type) {
                handleAddGrade(newGrade);
                // Reset state AFTER successful submission
                setNewGrade({ subjectId: '', type: '', date: formatDate(new Date()), marksObtained: '', totalMarks: 100 });
                setShowAddGrade(false);
            } else {
                setErrorMessage("Please fill in all required fields (Subject, Marks, and Exam Type).");
            }
        };

        const calculatePercentage = (marks, total) => {
            const m = parseFloat(marks);
            const t = parseFloat(total);
            if (isNaN(m) || isNaN(t) || t === 0) return 'N/A';
            return ((m / t) * 100).toFixed(1) + '%';
        };

        const groupedGrades = useMemo(() => {
            return grades.reduce((acc, grade) => {
                const subjectName = subjectOptions.find(s => s.value === grade.subjectId)?.label || 'Unknown Subject';
                if (!acc[subjectName]) {
                    acc[subjectName] = [];
                }
                acc[subjectName].push(grade);
                return acc;
            }, {});
        }, [grades, subjects]);


        return (
            <div className="p-4 space-y-6">
                <h2 className="text-2xl font-bold text-gray-800">Marks Bar (Grade Tracker)</h2>

                <button
                    onClick={() => setShowAddGrade(true)}
                    className="w-full py-3 bg-indigo-600 text-white font-semibold rounded-xl shadow-md hover:bg-indigo-700 transition-colors flex items-center justify-center space-x-2"
                >
                    <BarChart3 size={20} />
                    <span>Record New Grade</span>
                </button>

                <Modal 
                    isOpen={showAddGrade} 
                    onClose={() => { 
                        setShowAddGrade(false); 
                        // Reset state when closing modal to ensure fresh data next time
                        setNewGrade({ subjectId: '', type: '', date: formatDate(new Date()), marksObtained: '', totalMarks: 100 }); 
                    }} 
                    title="Record New Exam Score"
                >
                    {/* Render the stable GradeFormComponent here */}
                    <GradeFormComponent 
                        newGrade={newGrade}
                        handleGradeChange={handleGradeChange}
                        handleSubmitGrade={handleSubmitGrade}
                        subjectOptions={subjectOptions}
                    />
                </Modal>

                <div className="space-y-6">
                    {Object.keys(groupedGrades).length === 0 ? (
                         <p className="text-center text-gray-500 mt-8">No grades recorded yet.</p>
                    ) : (
                        Object.entries(groupedGrades).map(([subjectName, subjectGrades]) => (
                            <div key={subjectName} className="border border-gray-300 rounded-xl p-4 shadow-lg bg-white">
                                <h3 className="text-xl font-extrabold text-indigo-700 mb-3">{subjectName}</h3>
                                <div className="space-y-3">
                                    {subjectGrades.map(grade => (
                                        <div key={grade.id} className="flex items-center justify-between p-3 bg-gray-50 rounded-lg border">
                                            <div className="flex-1">
                                                <p className="font-semibold text-gray-800">{grade.type}</p>
                                                <p className="text-xs text-gray-500">Test Date: {new Date(grade.date).toLocaleDateString()}</p>
                                            </div>
                                            <div className="text-right flex items-center space-x-3">
                                                <span className="text-lg font-bold text-green-600">
                                                    {grade.marksObtained}/{grade.totalMarks} ({calculatePercentage(grade.marksObtained, grade.totalMarks)})
                                                </span>
                                                <button onClick={() => handleDeleteGrade(grade.id)} className="text-red-500 hover:text-red-700 p-1 rounded-full hover:bg-red-50" title="Delete Grade">
                                                    <Trash2 size={16} />
                                                </button>
                                            </div>
                                        </div>
                                    ))}
                                </div>
                            </div>
                        ))
                    )}
                </div>
            </div>
        );
    };

    const CreditHistoryView = () => (
        <div className="p-4 space-y-6">
            <h2 className="text-2xl font-bold text-gray-800">Credit Balance & History</h2>

            <div className="p-4 bg-amber-500 text-white rounded-xl shadow-lg text-center">
                <p className="text-sm font-semibold opacity-80">Current Credit Balance</p>
                <p className="text-4xl font-extrabold">${credit.balance.toFixed(2)}</p>
                
                {/* Reset Balance Button */}
                <button
                    onClick={() => setShowResetBalanceModal(true)}
                    className="mt-3 w-full py-2 bg-amber-600 text-white font-semibold rounded-lg hover:bg-amber-700 transition-colors flex items-center justify-center space-x-2"
                >
                    <RefreshCcw size={18}/> <span>Reset Current Balance</span>
                </button>
            </div>

            <ConfirmationModal
                isOpen={showResetBalanceModal}
                onClose={() => setShowResetBalanceModal(false)}
                onConfirm={handleManualBalanceReset}
                title="Confirm Balance Reset"
                message="Are you sure you want to permanently reset your current credit balance to $0.00? This action is logged but cannot be undone."
                confirmText="Reset Balance"
                confirmColor="bg-amber-600"
            />


            <div className="space-y-3">
                <h3 className="text-xl font-bold text-gray-800 border-b pb-2">Transaction History</h3>
                {credit.history.length === 0 ? (
                    <p className="text-gray-500">No transaction history yet.</p>
                ) : (
                    credit.history.slice().reverse().map((item, index) => (
                        <div 
                            // Using item.id for a stable key is essential for smooth list updates
                            key={item.id || index} 
                            className={`flex justify-between items-center p-3 rounded-lg border transition-colors ${
                                item.amount > 0 ? 'bg-green-50 border-green-200' : 'bg-red-50 border-red-200'
                            }`}
                        >
                            <div className="flex-1 min-w-0">
                                <p className="font-semibold text-gray-800 truncate">{item.note}</p>
                                <p className="text-xs text-gray-500">{new Date(item.date).toLocaleString()}</p>
                            </div>
                            <div className="flex items-center space-x-3 ml-4 flex-shrink-0">
                                <span className={`text-lg font-extrabold ${item.amount > 0 ? 'text-green-700' : 'text-red-700'}`}>
                                    {item.amount > 0 ? '+' : ''}${Math.abs(item.amount).toFixed(2)}
                                </span>
                                {/* Delete Button: uses item.id for reliable deletion */}
                                <button 
                                    onClick={() => handleDeleteCreditHistory(item.id)} 
                                    className="p-1 text-red-500 hover:text-red-700 rounded-full hover:bg-red-100"
                                    title="Delete History Item"
                                >
                                    <Trash2 size={16} />
                                </button>
                            </div>
                        </div>
                    ))
                )}
            </div>
        </div>
    );

    // --- Main App Layout ---
    return (
        <div className="min-h-screen bg-gray-100 font-sans flex flex-col">
            {/* Header */}
            <header className="p-4 bg-white shadow-lg sticky top-0 z-10 flex justify-between items-center">
                <h1 className="text-2xl font-black text-indigo-700">ProTasker</h1>
                <div className="p-2 px-4 bg-amber-500 text-white rounded-full shadow-md font-bold text-lg">
                    Credit: ${credit.balance.toFixed(2)}
                </div>
            </header>

            {/* Error Message Toast */}
            {errorMessage && (
                <div className="p-3 bg-red-500 text-white font-medium text-center">
                    <AlertTriangle size={18} className="inline-block mr-2"/> {errorMessage}
                </div>
            )}


            <main className="flex-grow container mx-auto p-0 pb-20">
                {view === 'tasks' && <FolderView />}
                {view === 'timer' && <TimerView />}
                {view === 'calendar' && <CalendarView />}
                {view === 'grades' && <GradesView />}
                {view === 'history' && <CreditHistoryView />}
            </main>

            {/* Navigation Bar */}
            <footer className="fixed bottom-0 left-0 right-0 bg-white border-t border-gray-200 shadow-2xl z-20">
                <div className="flex justify-around items-center p-2 max-w-lg mx-auto">
                    <IconWrapper onClick={() => setView('tasks')} title="Tasks" active={view === 'tasks'}>
                        <Folder size={24} />
                        <span className="text-xs mt-1">Tasks</span>
                    </IconWrapper>
                    <IconWrapper onClick={() => setView('timer')} title="Timer" active={view === 'timer'}>
                        <Clock size={24} />
                        <span className="text-xs mt-1">Timer</span>
                    </IconWrapper>
                    <IconWrapper onClick={() => setView('calendar')} title="Calendar" active={view === 'calendar'}>
                        <Calendar size={24} />
                        <span className="text-xs mt-1">Log</span>
                    </IconWrapper>
                    <IconWrapper onClick={() => setView('grades')} title="Marks" active={view === 'grades'}>
                        <BarChart3 size={24} />
                        <span className="text-xs mt-1">Marks</span>
                    </IconWrapper>
                    <IconWrapper onClick={() => setView('history')} title="Credit History" active={view === 'history'}>
                        <Award size={24} />
                        <span className="text-xs mt-1">Credit</span>
                    </IconWrapper>
                </div>
            </footer>
        </div>
    );
};

export default App;
