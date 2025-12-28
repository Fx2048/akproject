import React, { useState, useEffect, useRef } from 'react';
import { Play, Pause, Square, Plus, X, Calendar, Brain, Trophy, AlertCircle, Clock, CheckCircle, Circle, Flame, Heart, Coffee, Settings } from 'lucide-react';

const FocusFlowApp = () => {
  const [currentView, setCurrentView] = useState('dashboard');
  const [tasks, setTasks] = useState([]);
  const [motivationLevel, setMotivationLevel] = useState(5);
  const [burnoutMeter, setBurnoutMeter] = useState(50);
  const [dailyHours, setDailyHours] = useState(0);
  const [maxDailyHours, setMaxDailyHours] = useState(8);
  const [showAddTask, setShowAddTask] = useState(false);
  const [pomodoroActive, setPomodoroActive] = useState(false);
  const [pomodoroTime, setPomodoroTime] = useState(25 * 60);
  const [isBreak, setIsBreak] = useState(false);
  const [aiSchedule, setAiSchedule] = useState(null);
  const [generatingSchedule, setGeneratingSchedule] = useState(false);
  const [badges, setBadges] = useState([]);
  const [totalPoints, setTotalPoints] = useState(0);
  
  // Heart system from Pomodoro Vital
  const [hearts, setHearts] = useState(5);
  const [maxHearts, setMaxHearts] = useState(10);
  
  // Timer settings
  const [workMinutes, setWorkMinutes] = useState(25);
  const [breakMinutes, setBreakMinutes] = useState(5);
  const [showSettings, setShowSettings] = useState(false);
  
  // Google Calendar
  const [calendarEvents, setCalendarEvents] = useState([]);
  const [isLoadingCalendar, setIsLoadingCalendar] = useState(false);
  
  const audioContextRef = useRef(null);

  const [newTask, setNewTask] = useState({
    name: '',
    difficulty: 'medium',
    scheduledDate: new Date().toISOString().split('T')[0],
    scheduledTime: '09:00',
    estimatedHours: 2,
    status: 'pending'
  });

  // Sound system
  const playSound = (frequency, duration) => {
    if (!audioContextRef.current) {
      audioContextRef.current = new (window.AudioContext || window.webkitAudioContext)();
    }
    
    const ctx = audioContextRef.current;
    const oscillator = ctx.createOscillator();
    const gainNode = ctx.createGain();
    
    oscillator.connect(gainNode);
    gainNode.connect(ctx.destination);
    
    oscillator.frequency.value = frequency;
    oscillator.type = 'sine';
    
    gainNode.gain.setValueAtTime(0.3, ctx.currentTime);
    gainNode.gain.exponentialRampToValueAtTime(0.01, ctx.currentTime + duration);
    
    oscillator.start(ctx.currentTime);
    oscillator.stop(ctx.currentTime + duration);
  };

  // Pomodoro timer with sounds
  useEffect(() => {
    let interval;
    if (pomodoroActive && pomodoroTime > 0) {
      interval = setInterval(() => {
        setPomodoroTime(prev => prev - 1);
      }, 1000);
    } else if (pomodoroTime === 0) {
      setPomodoroActive(false);
      if (!isBreak) {
        // Break sound
        playSound(659.25, 0.15);
        setTimeout(() => playSound(783.99, 0.15), 150);
        setTimeout(() => playSound(880, 0.3), 300);
        setIsBreak(true);
        setPomodoroTime(breakMinutes * 60);
        alert('Great work! Time for a break! 🎉☕');
      } else {
        playSound(523.25, 0.2);
        setIsBreak(false);
        setPomodoroTime(workMinutes * 60);
        alert('Break over! Ready for the next Pomodoro session? 💪');
      }
    }
    return () => clearInterval(interval);
  }, [pomodoroActive, pomodoroTime, isBreak, workMinutes, breakMinutes]);

  // Calculate burnout meter
  useEffect(() => {
    const completedToday = tasks.filter(t => t.status === 'completed').length;
    const totalHoursWorked = tasks
      .filter(t => t.status === 'completed')
      .reduce((sum, t) => sum + t.estimatedHours, 0);
    
    setDailyHours(totalHoursWorked);
    
    let newMeter = 50;
    if (totalHoursWorked === 0 && completedToday === 0) {
      newMeter = 20;
    } else if (totalHoursWorked > maxDailyHours * 0.9) {
      newMeter = 90;
    } else if (totalHoursWorked >= maxDailyHours * 0.5 && totalHoursWorked <= maxDailyHours * 0.8) {
      newMeter = 50;
    } else if (totalHoursWorked < maxDailyHours * 0.3) {
      newMeter = 30;
    }
    
    setBurnoutMeter(newMeter);
  }, [tasks, maxDailyHours]);

  const getMeterColor = () => {
    if (burnoutMeter < 35) return 'from-blue-400 to-blue-600';
    if (burnoutMeter < 65) return 'from-green-400 to-green-600';
    return 'from-orange-400 to-red-600';
  };

  const getMeterLabel = () => {
    if (burnoutMeter < 35) return '😴 Lazy Zone';
    if (burnoutMeter < 65) return '✨ Balanced Flow';
    return '🔥 Burnout Risk!';
  };

  const addTask = () => {
    const task = {
      ...newTask,
      id: Date.now(),
      createdAt: new Date().toISOString()
    };
    setTasks([...tasks, task]);
    setShowAddTask(false);
    playSound(523.25, 0.2);
    setNewTask({
      name: '',
      difficulty: 'medium',
      scheduledDate: new Date().toISOString().split('T')[0],
      scheduledTime: '09:00',
      estimatedHours: 2,
      status: 'pending'
    });
  };

  const moveTask = (taskId, newStatus) => {
    setTasks(tasks.map(t => {
      if (t.id === taskId) {
        if (newStatus === 'completed' && t.status !== 'completed') {
          // Award points and hearts
          const points = t.difficulty === 'high' ? 100 : t.difficulty === 'medium' ? 60 : 30;
          setTotalPoints(prev => prev + points);
          setHearts(prev => Math.min(prev + 1, maxHearts));
          
          // Completion sound
          playSound(880, 0.1);
          setTimeout(() => playSound(1046.5, 0.15), 100);
          
          // Check for badges
          const completed = tasks.filter(t => t.status === 'completed').length + 1;
          if (completed === 1 && !badges.find(b => b.name === 'First Step')) {
            setBadges([...badges, { name: 'First Step', icon: '🌟' }]);
          }
          if (completed === 5 && !badges.find(b => b.name === 'Getting Started')) {
            setBadges([...badges, { name: 'Getting Started', icon: '🚀' }]);
          }
          if (completed === 10 && !badges.find(b => b.name === 'Consistent')) {
            setBadges([...badges, { name: 'Consistent', icon: '💪' }]);
          }
        }
        return { ...t, status: newStatus };
      }
      return t;
    }));
  };

  const deleteTask = (taskId) => {
    const task = tasks.find(t => t.id === taskId);
    if (!task.completed) {
      setHearts(prev => Math.max(prev - 1, 0));
      playSound(311.13, 0.2);
    }
    setTasks(tasks.filter(t => t.id !== taskId));
  };

  const connectGoogleCalendar = async () => {
    setIsLoadingCalendar(true);
    playSound(523.25, 0.1);
    
    // Simulated Google Calendar connection
    setTimeout(() => {
      const mockEvents = [
        { id: 1, title: 'Team Meeting', time: '10:00 AM', date: new Date().toISOString().split('T')[0] },
        { id: 2, title: 'Code Review', time: '2:00 PM', date: new Date().toISOString().split('T')[0] },
        { id: 3, title: 'Client Call', time: '4:30 PM', date: new Date().toISOString().split('T')[0] }
      ];
      setCalendarEvents(mockEvents);
      setIsLoadingCalendar(false);
      playSound(659.25, 0.15);
    }, 1500);
  };

  const generateAISchedule = async () => {
    setGeneratingSchedule(true);
    playSound(523.25, 0.2);
    
    try {
      const pendingTasks = tasks.filter(t => t.status === 'pending');
      
      const prompt = `You are an AI study planner for FocusFlow app. Generate a balanced daily schedule.

User Data:
- Motivation Level: ${motivationLevel}/10
- Max Daily Hours: ${maxDailyHours}
- Current Hearts: ${hearts}/${maxHearts}
- Tasks: ${JSON.stringify(pendingTasks.map(t => ({
  name: t.name,
  difficulty: t.difficulty,
  estimatedHours: t.estimatedHours,
  scheduledTime: t.scheduledTime
})))}
- Calendar Events: ${JSON.stringify(calendarEvents)}

Create a JSON schedule with:
1. Optimal task order (prioritize high-difficulty when motivation is high)
2. Break schedule using Pomodoro (${workMinutes}min work, ${breakMinutes}min break)
3. Avoid conflicts with calendar events
4. Risk assessment for burnout
5. Recommendations for balance

Respond ONLY with valid JSON:
{
  "schedule": [
    {
      "time": "09:00",
      "activity": "Task name or Break",
      "type": "task" or "break",
      "duration": hours,
      "pomodoros": number
    }
  ],
  "totalHours": number,
  "burnoutRisk": "low/medium/high",
  "recommendations": ["tip1", "tip2"]
}`;

      const response = await fetch('https://api.anthropic.com/v1/messages', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          model: 'claude-sonnet-4-20250514',
          max_tokens: 1000,
          messages: [{ role: 'user', content: prompt }]
        })
      });

      const data = await response.json();
      const aiResponse = data.content.find(c => c.type === 'text')?.text || '';
      
      const cleanedResponse = aiResponse.replace(/```json|```/g, '').trim();
      const schedule = JSON.parse(cleanedResponse);
      
      setAiSchedule(schedule);
      setCurrentView('schedule');
      playSound(880, 0.2);
    } catch (error) {
      console.error('AI Schedule Error:', error);
      alert('Could not generate schedule. Please try again.');
      playSound(311.13, 0.3);
    } finally {
      setGeneratingSchedule(false);
    }
  };

  const handlePomodoroStart = () => {
    setPomodoroActive(true);
    playSound(523.25, 0.2);
  };

  const handlePomodoroPause = () => {
    setPomodoroActive(false);
    playSound(392, 0.2);
  };

  const handlePomodoroStop = () => {
    setPomodoroActive(false);
    setPomodoroTime(isBreak ? breakMinutes * 60 : workMinutes * 60);
    playSound(261.63, 0.3);
  };

  const progress = ((isBreak ? breakMinutes * 60 : workMinutes * 60) - pomodoroTime) / 
                   (isBreak ? breakMinutes * 60 : workMinutes * 60) * 100;

  const TaskCard = ({ task }) => {
    const difficultyColors = {
      low: 'bg-green-100 border-green-300',
      medium: 'bg-yellow-100 border-yellow-300',
      high: 'bg-red-100 border-red-300'
    };

    const statusIcons = {
      pending: <Circle className="w-5 h-5 text-gray-400" />,
      'in-progress': <Clock className="w-5 h-5 text-blue-500" />,
      completed: <CheckCircle className="w-5 h-5 text-green-500" />
    };

    return (
      <div className={`p-4 rounded-lg border-2 ${difficultyColors[task.difficulty]} mb-3 shadow-sm`}>
        <div className="flex items-start justify-between mb-2">
          <div className="flex items-center gap-2">
            {statusIcons[task.status]}
            <h3 className="font-semibold text-gray-800">{task.name}</h3>
          </div>
          {task.status !== 'completed' && (
            <button onClick={() => deleteTask(task.id)} className="text-red-500 hover:text-red-700">
              <X className="w-4 h-4" />
            </button>
          )}
        </div>
        <div className="text-sm text-gray-600 space-y-1">
          <div>⏱️ {task.estimatedHours}h (~{Math.ceil(task.estimatedHours * 2)} Pomodoros)</div>
          <div>📅 {task.scheduledTime} on {task.scheduledDate}</div>
          <div className="capitalize">🎯 {task.difficulty} difficulty</div>
        </div>
        {task.status !== 'completed' && (
          <div className="flex gap-2 mt-3">
            {task.status === 'pending' && (
              <button
                onClick={() => moveTask(task.id, 'in-progress')}
                className="px-3 py-1 bg-blue-500 text-white rounded text-sm hover:bg-blue-600"
              >
                Start
              </button>
            )}
            {task.status === 'in-progress' && (
              <button
                onClick={() => moveTask(task.id, 'completed')}
                className="px-3 py-1 bg-green-500 text-white rounded text-sm hover:bg-green-600"
              >
                Complete
              </button>
            )}
          </div>
        )}
      </div>
    );
  };

  return (
    <div className="min-h-screen bg-gradient-to-br from-purple-900 via-indigo-900 to-blue-900 p-4">
      <div className="max-w-7xl mx-auto">
        {/* Enhanced Header with Hearts */}
        <div className="bg-white/10 backdrop-blur-md rounded-2xl shadow-lg p-6 mb-6 border border-white/20">
          <div className="flex items-center justify-between mb-4">
            <div className="flex items-center gap-3">
              <Brain className="w-8 h-8 text-purple-300" />
              <h1 className="text-3xl font-bold text-white">FocusFlow Vital</h1>
            </div>
            <div className="flex items-center gap-4">
              <button
                onClick={() => setShowSettings(!showSettings)}
                className="p-2 rounded-lg bg-white/20 hover:bg-white/30 transition-all"
              >
                <Settings className="text-white w-6 h-6" />
              </button>
              <div className="text-right">
                <div className="text-sm text-purple-200">Points</div>
                <div className="text-2xl font-bold text-yellow-300">{totalPoints}</div>
              </div>
              <Trophy className="w-8 h-8 text-yellow-400" />
            </div>
          </div>

          {/* Heart Bar */}
          <div className="mb-4 bg-white/5 rounded-xl p-4">
            <div className="flex items-center gap-4">
              <span className="text-white font-semibold">Life Energy:</span>
              <div className="flex gap-1 flex-1">
                {[...Array(maxHearts)].map((_, i) => (
                  <Heart
                    key={i}
                    className={`w-6 h-6 ${
                      i < hearts 
                        ? 'fill-red-500 text-red-500' 
                        : 'fill-gray-600 text-gray-600'
                    } transition-all duration-300`}
                  />
                ))}
              </div>
              <span className="text-white font-mono">{hearts}/{maxHearts}</span>
            </div>
          </div>

          {/* Burnout Meter */}
          <div className="mb-4">
            <div className="flex justify-between items-center mb-2">
              <span className="text-sm font-semibold text-purple-200">Energy Balance</span>
              <span className="text-sm font-bold text-white">{getMeterLabel()}</span>
            </div>
            <div className="relative h-8 bg-white/20 rounded-full overflow-hidden">
              <div
                className={`h-full bg-gradient-to-r ${getMeterColor()} transition-all duration-500 flex items-center justify-center`}
                style={{ width: `${burnoutMeter}%` }}
              >
                {burnoutMeter > 70 && <Flame className="w-5 h-5 text-white animate-pulse" />}
              </div>
            </div>
            <div className="flex justify-between text-xs text-purple-200 mt-1">
              <span>Lazy</span>
              <span>Balanced</span>
              <span>Burnout</span>
            </div>
          </div>

          {/* Daily Progress */}
          <div className="bg-purple-500/20 rounded-lg p-3 border border-purple-400/30">
            <div className="text-sm text-white">
              Today: <span className="font-bold">{dailyHours.toFixed(1)}h / {maxDailyHours}h</span>
              <span className="ml-3">Tasks: {tasks.filter(t => t.status === 'completed').length} completed</span>
            </div>
          </div>
        </div>

        {/* Settings Panel */}
        {showSettings && (
          <div className="bg-white/10 backdrop-blur-md rounded-2xl p-6 mb-6 border border-white/20">
            <h2 className="text-xl font-bold text-white mb-4">⚙️ Settings</h2>
            <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
              <div>
                <label className="text-white block mb-2">Work Time (minutes)</label>
                <input
                  type="number"
                  value={workMinutes}
                  onChange={(e) => {
                    setWorkMinutes(parseInt(e.target.value) || 1);
                    if (!pomodoroActive && !isBreak) setPomodoroTime(parseInt(e.target.value) * 60 || 60);
                  }}
                  className="w-full px-4 py-2 rounded-lg bg-white/20 text-white border border-white/30"
                  min="1"
                />
              </div>
              <div>
                <label className="text-white block mb-2">Break Time (minutes)</label>
                <input
                  type="number"
                  value={breakMinutes}
                  onChange={(e) => {
                    setBreakMinutes(parseInt(e.target.value) || 1);
                    if (!pomodoroActive && isBreak) setPomodoroTime(parseInt(e.target.value) * 60 || 60);
                  }}
                  className="w-full px-4 py-2 rounded-lg bg-white/20 text-white border border-white/30"
                  min="1"
                />
              </div>
            </div>
          </div>
        )}

        {/* Navigation */}
        <div className="flex gap-2 mb-6 bg-white/10 backdrop-blur-md rounded-lg p-2 shadow border border-white/20">
          {['dashboard', 'kanban', 'schedule', 'pomodoro', 'calendar'].map(view => (
            <button
              key={view}
              onClick={() => setCurrentView(view)}
              className={`flex-1 py-2 px-4 rounded-lg font-semibold capitalize transition-colors ${
                currentView === view
                  ? 'bg-purple-600 text-white'
                  : 'bg-white/10 text-purple-100 hover:bg-white/20'
              }`}
            >
              {view}
            </button>
          ))}
        </div>

        {/* Dashboard View */}
        {currentView === 'dashboard' && (
          <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
            <div className="bg-white/10 backdrop-blur-md rounded-xl shadow-lg p-6 border border-white/20">
              <h2 className="text-2xl font-bold text-white mb-4">Quick Start</h2>
              
              <div className="mb-4">
                <label className="block text-sm font-semibold text-purple-200 mb-2">
                  Current Motivation: {motivationLevel}/10
                </label>
                <input
                  type="range"
                  min="1"
                  max="10"
                  value={motivationLevel}
                  onChange={(e) => setMotivationLevel(Number(e.target.value))}
                  className="w-full"
                />
              </div>

              <div className="mb-4">
                <label className="block text-sm font-semibold text-purple-200 mb-2">
                  Max Daily Hours: {maxDailyHours}h
                </label>
                <input
                  type="range"
                  min="4"
                  max="12"
                  value={maxDailyHours}
                  onChange={(e) => setMaxDailyHours(Number(e.target.value))}
                  className="w-full"
                />
              </div>

              <div className="flex gap-3">
                <button
                  onClick={() => setShowAddTask(true)}
                  className="flex-1 bg-purple-600 text-white py-3 rounded-lg font-semibold hover:bg-purple-700 flex items-center justify-center gap-2 transition-all transform hover:scale-105"
                >
                  <Plus className="w-5 h-5" />
                  Add Task
                </button>
                <button
                  onClick={generateAISchedule}
                  disabled={tasks.filter(t => t.status === 'pending').length === 0 || generatingSchedule}
                  className="flex-1 bg-indigo-600 text-white py-3 rounded-lg font-semibold hover:bg-indigo-700 flex items-center justify-center gap-2 disabled:bg-gray-600 transition-all transform hover:scale-105"
                >
                  <Brain className="w-5 h-5" />
                  {generatingSchedule ? 'Generating...' : 'AI Schedule'}
                </button>
              </div>
            </div>

            {/* Badges */}
            <div className="bg-white/10 backdrop-blur-md rounded-xl shadow-lg p-6 border border-white/20">
              <h2 className="text-xl font-bold text-white mb-4">🏆 Achievements</h2>
              {badges.length > 0 ? (
                <div className="flex flex-wrap gap-3">
                  {badges.map((badge, i) => (
                    <div key={i} className="bg-yellow-500/20 border-2 border-yellow-400 rounded-lg px-4 py-2">
                      <span className="text-2xl mr-2">{badge.icon}</span>
                      <span className="font-semibold text-white">{badge.name}</span>
                    </div>
                  ))}
                </div>
              ) : (
                <p className="text-purple-200 text-center py-8">Complete tasks to earn badges!</p>
              )}
            </div>
          </div>
        )}

        {/* Kanban View */}
        {currentView === 'kanban' && (
          <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
            {['pending', 'in-progress', 'completed'].map(status => (
              <div key={status} className="bg-white/10 backdrop-blur-md rounded-xl shadow-lg p-4 border border-white/20">
                <h3 className="text-lg font-bold text-white mb-4 capitalize">{status.replace('-', ' ')}</h3>
                <div className="space-y-3">
                  {tasks.filter(t => t.status === status).map(task => (
                    <TaskCard key={task.id} task={task} />
                  ))}
                  {tasks.filter(t => t.status === status).length === 0 && (
                    <div className="text-center text-purple-300 py-8">No tasks</div>
                  )}
                </div>
              </div>
            ))}
          </div>
        )}

        {/* Schedule View */}
        {currentView === 'schedule' && (
          <div className="bg-white/10 backdrop-blur-md rounded-xl shadow-lg p-6 border border-white/20">
            <h2 className="text-2xl font-bold text-white mb-4">AI-Generated Schedule</h2>
            {aiSchedule ? (
              <div>
                <div className="mb-6 p-4 bg-purple-500/20 rounded-lg border border-purple-400/30">
                  <div className="font-semibold text-white mb-2">📊 Schedule Overview</div>
                  <div className="text-sm text-purple-100 space-y-1">
                    <div>Total Work Time: {aiSchedule.totalHours}h</div>
                    <div>Burnout Risk: <span className={`font-bold ${
                      aiSchedule.burnoutRisk === 'low' ? 'text-green-400' :
                      aiSchedule.burnoutRisk === 'medium' ? 'text-yellow-400' : 'text-red-400'
                    }`}>{aiSchedule.burnoutRisk.toUpperCase()}</span></div>
                  </div>
                </div>

                <div className="space-y-3 mb-6">
                  {aiSchedule.schedule.map((item, i) => (
                    <div key={i} className={`p-4 rounded-lg ${
                      item.type === 'break' ? 'bg-green-500/20 border-2 border-green-400' : 'bg-blue-500/20 border-2 border-blue-400'
                    }`}>
                      <div className="flex items-center justify-between">
                        <div>
                          <div className="font-semibold text-white">{item.time} - {item.activity}</div>
                          <div className="text-sm text-purple-200">
                            {item.type === 'task' && `${item.duration}h (${item.pomodoros} Pomodoros)`}
                            {item.type === 'break' && '5-15 min break'}
                          </div>
                        </div>
                        {item.type === 'task' ? <Calendar className="w-5 h-5 text-blue-300" /> : <Coffee className="w-5 h-5 text-green-300" />}
                      </div>
                    </div>
                  ))}
                </div>

                <div className="bg-yellow-500/20 border-2 border-yellow-400 rounded-lg p-4">
                  <div className="font-semibold text-white mb-2">💡 AI Recommendations:</div>
                  <ul className="text-sm text-purple-100 space-y-1">
                    {aiSchedule.recommendations.map((rec, i) => (
                      <li key={i}>• {rec}</li>
                    ))}
                  </ul>
                </div>
              </div>
            ) : (
              <div className="text-center py-12 text-purple-300">
                <Calendar className="w-16 h-16 mx-auto mb-4 opacity-50" />
                <p>Generate an AI schedule from the Dashboard</p>
              </div>
            )}
          </div>
        )}

        {/* Pomodoro View */}
        {currentView === 'pomodoro' && (
          <div className="bg-white/10 backdrop-blur-md rounded-xl shadow-lg p-8 border border-white/20 text-center">
            <h2 className="text-2xl font-bold text-white mb-4">
              {isBreak ? '☕ Break Time' : '🎯 Focus Time'}
            </h2>
            <div className="text-8xl font-bold text-white mb-6 font-mono">
              {Math.floor(pomodoroTime / 60).toString().padStart(2, '0')}:
              {(pomodoroTime % 60).toString().padStart(2, '0')}
            </div>
            
            {/* Progress Bar */}
            <div className="w-full h-4 bg-white/20 rounded-full overflow-hidden mb-8">
              <div
                className={`h-full transition-all duration-1000 ${
                  isBreak ? 'bg-green-400' : 'bg-orange-400'
                }`}
                style={{ width: `${progress}%` }}
              />
            </div>

            <div className="flex justify-center gap-4">
              {!pomodoroActive ? (
                <button
                  onClick={handlePomodoroStart}
                  className="px-8 py-4 bg-green-500 hover:bg-green-600 text-white rounded-full font-semibold text-lg flex items-center gap-2 transition-all transform hover:scale-105"
                >
                  <Play className="w-6 h-6" />
                  Start
                </button>
              ) : (
                <button
                  onClick={handlePomodoroPause}
                  className="px-8 py-4 bg-yellow-500 hover:bg-yellow-600 text-white rounded-full font-semibold text-lg flex items-center gap-2 transition-all transform hover:scale-105"
                >
                  <Pause className="w-6 h-6" />
                  Pause
                </button>
              )}
              <button
                onClick={handlePomodoroStop}
                className="px-8 py-4 bg-red-500 hover:bg-red-600 text-white rounded-full font-semibold text-lg flex items-center gap-2 transition-all transform hover:scale-105"
              >
                <Square className="w-6 h-6" />
                Stop
              </button>
            </div>
            
            <div className="mt-6 text-purple-200">
              <p className="text-sm">
                {isBreak 
                  ? 'Take a break! Stretch, hydrate, or take a short walk.' 
                  : 'Stay focused on your current task. You got this!'}
              </p>
            </div>
          </div>
        )}

        {/* Calendar View */}
        {currentView === 'calendar' && (
          <div className="bg-white/10 backdrop-blur-md rounded-xl shadow-lg p-6 border border-white/20">
            <div className="flex justify-between items-center mb-6">
              <h2 className="text-2xl font-bold text-white flex items-center gap-2">
                <Calendar className="text-blue-300" />
                Google Calendar Integration
              </h2>
              <button
                onClick={connectGoogleCalendar}
                disabled={isLoadingCalendar}
                className="px-6 py-3 bg-blue-500 hover:bg-blue-600 disabled:bg-gray-500 text-white rounded-lg font-semibold transition-all transform hover:scale-105 flex items-center gap-2"
              >
                {isLoadingCalendar ? (
                  <>
                    <div className="animate-spin rounded-full h-5 w-5 border-b-2 border-white"></div>
                    Connecting...
                  </>
                ) : (
                  'Connect Calendar'
                )}
              </button>
            </div>
            
            {calendarEvents.length > 0 ? (
              <div className="space-y-3">
                {calendarEvents.map(event => (
                  <div key={event.id} className="bg-white/10 p-4 rounded-lg border border-white/20 hover:bg-white/20 transition-all">
                    <div className="flex items-center gap-3">
                      <Calendar className="w-5 h-5 text-blue-300" />
                      <div className="flex-1">
                        <div className="text-white font-semibold">{event.title}</div>
                        <div className="text-blue-300 text-sm">{event.time} • {event.date}</div>
                      </div>
                    </div>
                  </div>
                ))}
              </div>
            ) : (
              <div className="text-center py-16">
                <Calendar className="w-20 h-20 mx-auto mb-4 text-purple-300 opacity-50" />
                <p className="text-purple-200 text-lg mb-2">No calendar connected</p>
                <p className="text-purple-300 text-sm">Connect your Google Calendar to sync events with your schedule</p>
              </div>
            )}
          </div>
        )}

        {/* Add Task Modal */}
        {showAddTask && (
          <div className="fixed inset-0 bg-black bg-opacity-70 flex items-center justify-center p-4 z-50">
            <div className="bg-gradient-to-br from-purple-800 to-indigo-900 rounded-2xl shadow-2xl p-6 max-w-md w-full border border-purple-400">
              <div className="flex justify-between items-center mb-4">
                <h3 className="text-2xl font-bold text-white">Add New Task</h3>
                <button onClick={() => setShowAddTask(false)} className="text-purple-200 hover:text-white">
                  <X className="w-6 h-6" />
                </button>
              </div>
              
              <div className="space-y-4">
                <div>
                  <label className="block text-sm font-semibold text-purple-200 mb-1">Task Name</label>
                  <input
                    type="text"
                    value={newTask.name}
                    onChange={(e) => setNewTask({...newTask, name: e.target.value})}
                    className="w-full px-4 py-2 rounded-lg bg-white/20 text-white border border-white/30 placeholder-purple-300 focus:ring-2 focus:ring-purple-400"
                    placeholder="e.g., Study Chapter 5"
                  />
                </div>

                <div>
                  <label className="block text-sm font-semibold text-purple-200 mb-1">Difficulty</label>
                  <select
                    value={newTask.difficulty}
                    onChange={(e) => setNewTask({...newTask, difficulty: e.target.value})}
                    className="w-full px-4 py-2 rounded-lg bg-white/20 text-white border border-white/30 focus:ring-2 focus:ring-purple-400"
                  >
                    <option value="low">Low</option>
                    <option value="medium">Medium</option>
                    <option value="high">High</option>
                  </select>
                </div>

                <div>
                  <label className="block text-sm font-semibold text-purple-200 mb-1">Estimated Hours</label>
                  <input
                    type="number"
                    min="0.5"
                    max="8"
                    step="0.5"
                    value={newTask.estimatedHours}
                    onChange={(e) => setNewTask({...newTask, estimatedHours: parseFloat(e.target.value)})}
                    className="w-full px-4 py-2 rounded-lg bg-white/20 text-white border border-white/30 focus:ring-2 focus:ring-purple-400"
                  />
                </div>

                <div className="grid grid-cols-2 gap-4">
                  <div>
                    <label className="block text-sm font-semibold text-purple-200 mb-1">Date</label>
                    <input
                      type="date"
                      value={newTask.scheduledDate}
                      onChange={(e) => setNewTask({...newTask, scheduledDate: e.target.value})}
                      className="w-full px-4 py-2 rounded-lg bg-white/20 text-white border border-white/30 focus:ring-2 focus:ring-purple-400"
                    />
                  </div>
                  <div>
                    <label className="block text-sm font-semibold text-purple-200 mb-1">Time</label>
                    <input
                      type="time"
                      value={newTask.scheduledTime}
                      onChange={(e) => setNewTask({...newTask, scheduledTime: e.target.value})}
                      className="w-full px-4 py-2 rounded-lg bg-white/20 text-white border border-white/30 focus:ring-2 focus:ring-purple-400"
                    />
                  </div>
                </div>

                <button
                  onClick={addTask}
                  disabled={!newTask.name}
                  className="w-full bg-purple-600 text-white py-3 rounded-lg font-semibold hover:bg-purple-700 disabled:bg-gray-600 transition-all transform hover:scale-105"
                >
                  Add Task
                </button>
              </div>
            </div>
          </div>
        )}

        {/* Burnout Warning */}
        {burnoutMeter > 80 && (
          <div className="fixed bottom-4 right-4 bg-red-500 text-white p-4 rounded-lg shadow-lg max-w-sm animate-pulse z-50 border-2 border-red-300">
            <div className="flex items-center gap-2 mb-2">
              <AlertCircle className="w-6 h-6" />
              <span className="font-bold">Burnout Alert!</span>
            </div>
            <p className="text-sm">You're working too hard! Take a mandatory break to recharge. 🧘‍♀️</p>
          </div>
        )}
      </div>
    </div>
  );
};

export default FocusFlowApp;
