import React, { useState, useEffect } from 'react';
import { AlertCircle, TrendingUp, Target, CheckCircle, Clock, Brain, BarChart3, Zap, Heart, RefreshCw, Loader, AlertTriangle } from 'lucide-react';

const RejectionEngine = () => {
  const [activeTab, setActiveTab] = useState('input');
  const [rejections, setRejections] = useState([]);
  const [analyzing, setAnalyzing] = useState(false);
  const [currentInput, setCurrentInput] = useState('');
  const [insights, setInsights] = useState(null);
  const [error, setError] = useState(null);
  const [userProfile, setUserProfile] = useState({
    name: "Alex Chen",
    targetRole: "Software Engineer",
    skillGaps: [],
    improvementTracking: []
  });

  useEffect(() => {
    loadFromStorage();
  }, []);

  const loadFromStorage = async () => {
    try {
      const stored = await window.storage.get('rejection_engine_data');
      if (stored && stored.value) {
        const data = JSON.parse(stored.value);
        setRejections(data.rejections || []);
        setInsights(data.insights || null);
        setUserProfile(data.profile || userProfile);
      }
    } catch (err) {
      console.log('No stored data found, starting fresh');
    }
  };

  const saveToStorage = async (newRejections, newInsights, newProfile) => {
    try {
      await window.storage.set('rejection_engine_data', JSON.stringify({
        rejections: newRejections,
        insights: newInsights,
        profile: newProfile
      }));
    } catch (err) {
      console.error('Failed to save data:', err);
    }
  };

  const analyzeRejectionWithClaude = async (text) => {
    setAnalyzing(true);
    setError(null);
    
    try {
      const response = await fetch("https://api.anthropic.com/v1/messages", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify({
          model: "claude-sonnet-4-20250514",
          max_tokens: 1000,
          messages: [{
            role: "user",
            content: `Analyze this job rejection feedback and extract structured information.

Rejection text: "${text}"

Extract and return ONLY a JSON object with this exact structure (no markdown, no preamble):
{
  "company": "company name or 'Unknown' if not mentioned",
  "role": "job title or 'Software Engineer' if not specified",
  "stage": "one of: resume_screen, phone, technical, behavioral, final",
  "explicitReason": "the main reason given for rejection",
  "implicitSignals": ["array", "of", "implied", "issues"],
  "severity": "one of: low, medium, high"
}`
          }]
        })
      });

      if (!response.ok) {
        throw new Error(`API error: ${response.status}`);
      }

      const data = await response.json();
      const textContent = data.content.find(item => item.type === "text")?.text || "";
      
      const cleanedText = textContent.replace(/```json\n?/g, '').replace(/```\n?/g, '').trim();
      const parsed = JSON.parse(cleanedText);

      const newRejection = {
        id: Date.now(),
        company: parsed.company,
        role: parsed.role,
        stage: parsed.stage,
        explicitReason: parsed.explicitReason,
        implicitSignals: parsed.implicitSignals || [],
        timestamp: new Date().toISOString(),
        emotionalContext: parsed.stage === 'final' ? 'devastating' : parsed.stage === 'resume_screen' ? 'expected' : 'surprising',
        rawText: text,
        severity: parsed.severity
      };

      const updatedRejections = [...rejections, newRejection];
      setRejections(updatedRejections);

      const newInsights = generateInsights(updatedRejections);
      setInsights(newInsights);

      const updatedProfile = updateProfileFromRejection(userProfile, newRejection);
      setUserProfile(updatedProfile);

      await saveToStorage(updatedRejections, newInsights, updatedProfile);
      
      setCurrentInput('');
      setActiveTab('insights');
    } catch (err) {
      console.error('Analysis error:', err);
      setError(err.message || 'Failed to analyze rejection. Please try again.');
    } finally {
      setAnalyzing(false);
    }
  };

  const generateInsights = (rejectionList) => {
    const stageCount = {};
    const reasonCount = {};

    rejectionList.forEach(r => {
      stageCount[r.stage] = (stageCount[r.stage] || 0) + 1;
      reasonCount[r.explicitReason] = (reasonCount[r.explicitReason] || 0) + 1;
    });

    const topStage = Object.entries(stageCount).sort((a, b) => b[1] - a[1])[0];
    const topReason = Object.entries(reasonCount).sort((a, b) => b[1] - a[1])[0];

    const patterns = [];
    if (topStage && topStage[1] >= 2) {
      patterns.push({
        type: 'stage_pattern',
        message: `You've been rejected at ${topStage[0].replace('_', ' ')} stage ${topStage[1]} times`,
        severity: 'high',
        actionable: true
      });
    }

    if (topReason && topReason[1] >= 2) {
      patterns.push({
        type: 'skill_gap',
        message: `"${topReason[0]}" mentioned in ${topReason[1]} rejections`,
        severity: 'critical',
        actionable: true
      });
    }

    const recentRejections = rejectionList.slice(-5);
    const techStageRejections = recentRejections.filter(r => r.stage === 'technical').length;
    if (techStageRejections >= 3) {
      patterns.push({
        type: 'interview_prep',
        message: 'Technical interview performance needs improvement',
        severity: 'high',
        actionable: true
      });
    }

    return {
      totalRejections: rejectionList.length,
      patterns,
      stageBreakdown: stageCount,
      topReasons: Object.entries(reasonCount).sort((a, b) => b[1] - a[1]).slice(0, 3),
      improvementRate: calculateImprovementRate(rejectionList),
      nextActions: generateActionPlan(patterns, rejectionList)
    };
  };

  const calculateImprovementRate = (rejectionList) => {
    if (rejectionList.length < 4) return null;
    
    const first = rejectionList.slice(0, Math.floor(rejectionList.length / 2));
    const second = rejectionList.slice(Math.floor(rejectionList.length / 2));
    
    const firstInterviewRate = first.filter(r => r.stage !== 'resume_screen').length / first.length;
    const secondInterviewRate = second.filter(r => r.stage !== 'resume_screen').length / second.length;
    
    return ((secondInterviewRate - firstInterviewRate) * 100).toFixed(1);
  };

  const generateActionPlan = (patterns) => {
    const actions = [];
    
    patterns.forEach(pattern => {
      if (pattern.type === 'skill_gap') {
        if (pattern.message.toLowerCase().includes('system design')) {
          actions.push({
            priority: 'critical',
            action: 'Complete System Design Course',
            description: 'Take "Grokking System Design" course',
            timeframe: '2 weeks',
            impact: 'Addresses 40% of rejections',
            resources: ['educative.io/system-design', 'YouTube: System Design Primer']
          });
        } else if (pattern.message.toLowerCase().includes('algorithm')) {
          actions.push({
            priority: 'critical',
            action: 'LeetCode Intensive Training',
            description: 'Solve 50 medium problems focusing on patterns',
            timeframe: '4 weeks',
            impact: 'Direct preparation for technical screens'
          });
        } else if (pattern.message.toLowerCase().includes('production') || pattern.message.toLowerCase().includes('experience')) {
          actions.push({
            priority: 'high',
            action: 'Deploy Production-Grade Application',
            description: 'Build and deploy app with CI/CD, monitoring, and real users',
            timeframe: '3 weeks',
            impact: 'Demonstrates production experience'
          });
        }
      }
      
      if (pattern.type === 'stage_pattern' && pattern.message.includes('technical')) {
        actions.push({
          priority: 'critical',
          action: 'Mock Interview Practice',
          description: 'Schedule 5 mock technical interviews on Pramp',
          timeframe: '2 weeks',
          impact: 'Improves interview performance and confidence'
        });
      }
    });

    if (actions.length === 0) {
      actions.push({
        priority: 'medium',
        action: 'Expand Application Pool',
        description: 'Apply to 15 more positions matching your profile',
        timeframe: '1 week',
        impact: 'Increases opportunities and gathers more data'
      });
    }

    return actions;
  };

  const updateProfileFromRejection = (profile, rejection) => {
    const skillGaps = [...profile.skillGaps];
    const gapIndex = skillGaps.findIndex(g => g.area === rejection.explicitReason);
    
    if (gapIndex >= 0) {
      skillGaps[gapIndex].evidenceCount += 1;
      skillGaps[gapIndex].lastOccurrence = rejection.timestamp;
    } else {
      skillGaps.push({
        area: rejection.explicitReason,
        evidenceCount: 1,
        lastOccurrence: rejection.timestamp,
        remediationStatus: 'identified',
        priority: rejection.stage === 'technical' || rejection.stage === 'final' ? 'high' : 'medium'
      });
    }

    return { ...profile, skillGaps };
  };

  const startRemediationTracking = async (gap) => {
    const updated = {
      ...userProfile,
      skillGaps: userProfile.skillGaps.map(g => 
        g.area === gap.area ? { ...g, remediationStatus: 'in_progress', startedDate: new Date().toISOString() } : g
      ),
      improvementTracking: [
        ...userProfile.improvementTracking,
        {
          gap: gap.area,
          startDate: new Date().toISOString(),
          actions: [],
          status: 'active'
        }
      ]
    };
    setUserProfile(updated);
    await saveToStorage(rejections, insights, updated);
  };

  const RejectionInput = () => (
    <div className="space-y-6">
      <div className="bg-gradient-to-r from-blue-900/30 to-purple-900/30 border border-blue-700 rounded-xl p-6 backdrop-blur-sm">
        <div className="flex items-start gap-4">
          <div className="bg-blue-500 p-3 rounded-lg">
            <Brain className="w-6 h-6 text-white" />
          </div>
          <div>
            <h3 className="font-semibold text-blue-300 text-lg">AI-Powered Rejection Analysis</h3>
            <p className="text-sm text-gray-300 mt-2 leading-relaxed">
              Paste your rejection email or interview feedback. Claude will extract structured insights and identify patterns.
            </p>
          </div>
        </div>
      </div>

      {error && (
        <div className="bg-red-900/30 border border-red-700 rounded-xl p-6 backdrop-blur-sm animate-pulse">
          <div className="flex items-start gap-4">
            <div className="bg-red-500 p-3 rounded-lg">
              <AlertTriangle className="w-6 h-6 text-white" />
            </div>
            <div>
              <h3 className="font-semibold text-red-300 text-lg">Analysis Error</h3>
              <p className="text-sm text-red-200 mt-2">{error}</p>
            </div>
          </div>
        </div>
      )}

      <div>
        <label className="block text-sm font-medium text-gray-300 mb-3">
          Rejection Details
        </label>
        <textarea
          value={currentInput}
          onChange={(e) => setCurrentInput(e.target.value)}
          placeholder="Paste rejection email or describe the feedback you received...

Example: 'Thank you for interviewing with Google. Unfortunately, we've decided to move forward with other candidates who have more production experience with distributed systems and stronger algorithmic problem-solving skills.'"
          className="w-full h-48 px-4 py-3 bg-gray-800 border border-gray-600 text-gray-100 placeholder-gray-500 rounded-xl focus:ring-2 focus:ring-blue-500 focus:border-transparent transition-all"
        />
      </div>

      <div className="flex gap-3">
        <button
          onClick={() => analyzeRejectionWithClaude(currentInput)}
          disabled={!currentInput.trim() || analyzing}
          className="flex-1 bg-gradient-to-r from-blue-600 to-purple-600 text-white py-4 rounded-xl font-semibold hover:from-blue-700 hover:to-purple-700 disabled:from-gray-700 disabled:to-gray-700 disabled:cursor-not-allowed flex items-center justify-center gap-3 transition-all transform hover:scale-105 disabled:hover:scale-100 shadow-lg"
        >
          {analyzing ? (
            <>
              <Loader className="w-6 h-6 animate-spin" />
              Analyzing with Claude AI...
            </>
          ) : (
            <>
              <Zap className="w-6 h-6" />
              Analyze Rejection
            </>
          )}
        </button>
      </div>

      {rejections.length > 0 && (
        <div className="mt-8 bg-gray-800 rounded-xl p-6 border border-gray-700">
          <h4 className="font-semibold text-gray-200 mb-4 flex items-center gap-2">
            <Clock className="w-5 h-5 text-blue-400" />
            Recent Rejections ({rejections.length})
          </h4>
          <div className="space-y-3 max-h-64 overflow-y-auto pr-2">
            {rejections.slice(-5).reverse().map(r => (
              <div key={r.id} className="bg-gray-900 p-4 rounded-lg border border-gray-700 hover:border-gray-600 transition-all">
                <div className="flex justify-between items-start">
                  <div>
                    <span className="font-medium text-gray-100">{r.company}</span>
                    <span className="text-gray-400 text-sm ml-2">• {r.role}</span>
                  </div>
                  <span className={`text-xs px-3 py-1 rounded-full font-medium ${
                    r.severity === 'high' ? 'bg-red-900 text-red-300' :
                    r.severity === 'medium' ? 'bg-yellow-900 text-yellow-300' :
                    'bg-gray-700 text-gray-300'
                  }`}>
                    {r.stage.replace('_', ' ')}
                  </span>
                </div>
                <p className="text-sm text-gray-400 mt-2">{r.explicitReason}</p>
              </div>
            ))}
          </div>
        </div>
      )}
    </div>
  );

  const InsightsDashboard = () => {
    if (!insights) {
      return (
        <div className="text-center py-16 text-gray-400">
          <div className="bg-gray-800 rounded-full w-24 h-24 flex items-center justify-center mx-auto mb-6">
            <BarChart3 className="w-12 h-12 opacity-50" />
          </div>
          <p className="text-lg">No insights yet. Add rejection data to see patterns.</p>
        </div>
      );
    }

    return (
      <div className="space-y-6">
        <div className="grid grid-cols-3 gap-4">
          <div className="bg-gradient-to-br from-red-900/40 to-red-800/40 p-6 rounded-xl border border-red-700 backdrop-blur-sm">
            <div className="text-3xl font-bold text-red-300">{insights.totalRejections}</div>
            <div className="text-sm text-red-400 mt-1">Total Rejections</div>
          </div>
          <div className="bg-gradient-to-br from-blue-900/40 to-blue-800/40 p-6 rounded-xl border border-blue-700 backdrop-blur-sm">
            <div className="text-3xl font-bold text-blue-300">
              {insights.patterns.filter(p => p.actionable).length}
            </div>
            <div className="text-sm text-blue-400 mt-1">Actionable Patterns</div>
          </div>
          <div className="bg-gradient-to-br from-green-900/40 to-green-800/40 p-6 rounded-xl border border-green-700 backdrop-blur-sm">
            <div className="text-3xl font-bold text-green-300">
              {insights.improvementRate ? `+${insights.improvementRate}%` : 'N/A'}
            </div>
            <div className="text-sm text-green-400 mt-1">Interview Rate Δ</div>
          </div>
        </div>

        {insights.patterns.length > 0 && (
          <div className="bg-yellow-900/30 border-l-4 border-yellow-500 p-6 rounded-xl backdrop-blur-sm">
            <div className="flex items-start gap-4">
              <div className="bg-yellow-500 p-3 rounded-lg">
                <AlertCircle className="w-6 h-6 text-white" />
              </div>
              <div className="flex-1">
                <h3 className="font-semibold text-yellow-300 mb-4 text-lg">Detected Patterns</h3>
                <div className="space-y-3">
                  {insights.patterns.map((pattern, idx) => (
                    <div key={idx} className="bg-gray-800 p-4 rounded-lg border border-gray-700">
                      <div className="flex items-start justify-between gap-4">
                        <div className="flex-1">
                          <p className="font-medium text-gray-100">{pattern.message}</p>
                          <p className="text-sm text-gray-400 mt-2">
                            {pattern.type === 'skill_gap' && 'Root cause identified: Missing technical competency'}
                            {pattern.type === 'stage_pattern' && 'Preparation focus area detected'}
                            {pattern.type === 'interview_prep' && 'Interview performance needs attention'}
                          </p>
                        </div>
                        <span className={`text-xs px-3 py-1 rounded-full whitespace-nowrap font-medium ${
                          pattern.severity === 'critical' ? 'bg-red-900 text-red-300' :
                          pattern.severity === 'high' ? 'bg-orange-900 text-orange-300' :
                          'bg-yellow-900 text-yellow-300'
                        }`}>
                          {pattern.severity}
                        </span>
                      </div>
                    </div>
                  ))}
                </div>
              </div>
            </div>
          </div>
        )}

        <div className="bg-gray-800 border border-gray-700 rounded-xl p-6">
          <h3 className="font-semibold text-gray-200 mb-4 flex items-center gap-2">
            <Target className="w-5 h-5 text-blue-400" />
            Rejection Stage Breakdown
          </h3>
          <div className="space-y-3">
            {Object.entries(insights.stageBreakdown).map(([stage, count]) => {
              const percentage = (count / insights.totalRejections * 100).toFixed(0);
              return (
                <div key={stage}>
                  <div className="flex justify-between text-sm mb-2">
                    <span className="capitalize text-gray-300">{stage.replace('_', ' ')}</span>
                    <span className="font-medium text-gray-200">{count} ({percentage}%)</span>
                  </div>
                  <div className="w-full bg-gray-700 rounded-full h-3 overflow-hidden">
                    <div 
                      className="bg-gradient-to-r from-blue-500 to-purple-500 h-3 rounded-full transition-all duration-500"
                      style={{ width: `${percentage}%` }}
                    />
                  </div>
                </div>
              );
            })}
          </div>
        </div>

        <div className="bg-gray-800 border border-gray-700 rounded-xl p-6">
          <h3 className="font-semibold text-gray-200 mb-4">Top Rejection Reasons</h3>
          <div className="space-y-2">
            {insights.topReasons.map(([reason, count], idx) => (
              <div key={idx} className="flex items-center justify-between p-3 bg-gray-900 rounded-lg hover:bg-gray-850 transition-all">
                <span className="text-sm text-gray-300">{reason}</span>
                <span className="bg-gray-700 text-gray-300 px-3 py-1 rounded-full text-xs font-medium">
                  {count}x
                </span>
              </div>
            ))}
          </div>
        </div>
      </div>
    );
  };

  const ActionPlan = () => {
    if (!insights || insights.nextActions.length === 0) {
      return (
        <div className="text-center py-12 text-gray-500">
          <CheckCircle className="w-16 h-16 mx-auto mb-4 opacity-50" />
          <p>No action items yet. Analyze rejections to get personalized recommendations.</p>
        </div>
      );
    }

    return (
      <div className="space-y-6">
        <div className="bg-gradient-to-r from-purple-50 to-blue-50 border border-purple-200 rounded-lg p-4">
          <div className="flex items-start gap-3">
            <TrendingUp className="w-6 h-6 text-purple-600 mt-0.5" />
            <div>
              <h3 className="font-semibold text-purple-900">Personalized Action Roadmap</h3>
              <p className="text-sm text-purple-700 mt-1">
                Based on {insights.totalRejections} rejections, here's your data-driven improvement plan
              </p>
            </div>
          </div>
        </div>

        <div className="space-y-4">
          {insights.nextActions.map((action, idx) => (
            <div key={idx} className="bg-white border-2 border-gray-200 rounded-lg p-5 hover:border-blue-300 transition-colors">
              <div className="flex items-start justify-between mb-3">
                <div className="flex items-start gap-3 flex-1">
                  <div className={`w-8 h-8 rounded-full flex items-center justify-center text-white font-bold flex-shrink-0 ${
                    action.priority === 'critical' ? 'bg-red-500' :
                    action.priority === 'high' ? 'bg-orange-500' :
                    'bg-blue-500'
                  }`}>
                    {idx + 1}
                  </div>
                  <div className="flex-1 min-w-0">
                    <h4 className="font-semibold text-gray-900 text-lg">{action.action}</h4>
                    <p className="text-gray-600 mt-1">{action.description}</p>
                  </div>
                </div>
                <span className={`px-3 py-1 rounded-full text-xs font-medium whitespace-nowrap ml-2 ${
                  action.priority === 'critical' ? 'bg-red-100 text-red-700' :
                  action.priority === 'high' ? 'bg-orange-100 text-orange-700' :
                  'bg-blue-100 text-blue-700'
                }`}>
                  {action.priority}
                </span>
              </div>

              <div className="grid grid-cols-2 gap-4 mt-4 text-sm">
                <div className="flex items-center gap-2 text-gray-600">
                  <Clock className="w-4 h-4" />
                  <span>{action.timeframe}</span>
                </div>
                <div className="flex items-center gap-2 text-gray-600">
                  <Target className="w-4 h-4" />
                  <span>{action.impact}</span>
                </div>
              </div>

              {action.resources && (
                <div className="mt-4 pt-4 border-t border-gray-200">
                  <p className="text-xs font-medium text-gray-500 mb-2">RESOURCES</p>
                  <div className="flex flex-wrap gap-2">
                    {action.resources.map((resource, ridx) => (
                      <span key={ridx} className="text-xs bg-gray-100 text-gray-700 px-2 py-1 rounded">
                        {resource}
                      </span>
                    ))}
                  </div>
                </div>
              )}

              <button className="w-full mt-4 bg-blue-600 text-white py-2 rounded-lg font-medium hover:bg-blue-700 transition-colors">
                Start This Action
              </button>
            </div>
          ))}
        </div>

        <div className="bg-green-50 border border-green-200 rounded-lg p-4">
          <div className="flex items-start gap-3">
            <Heart className="w-5 h-5 text-green-600 mt-0.5" />
            <div>
              <h4 className="font-semibold text-green-900">Remember</h4>
              <p className="text-sm text-green-700 mt-1">
                Rejection is feedback, not failure. You're building data to optimize your strategy. 
                Every rejection brings you closer to the right opportunity.
              </p>
            </div>
          </div>
        </div>
      </div>
    );
  };

  const ProfileTracker = () => {
    return (
      <div className="space-y-6">
        <div className="bg-gradient-to-r from-indigo-50 to-purple-50 border border-indigo-200 rounded-lg p-4">
          <h3 className="font-semibold text-indigo-900 mb-1">{userProfile.name}</h3>
          <p className="text-sm text-indigo-700">Target Role: {userProfile.targetRole}</p>
        </div>

        <div>
          <h3 className="font-semibold text-gray-900 mb-3 flex items-center gap-2">
            <AlertCircle className="w-5 h-5" />
            Identified Skill Gaps ({userProfile.skillGaps.length})
          </h3>
          
          {userProfile.skillGaps.length === 0 ? (
            <div className="text-center py-8 text-gray-500">
              <CheckCircle className="w-12 h-12 mx-auto mb-3 opacity-50" />
              <p>No skill gaps identified yet</p>
            </div>
          ) : (
            <div className="space-y-3">
              {userProfile.skillGaps.map((gap, idx) => (
                <div key={idx} className="bg-white border border-gray-200 rounded-lg p-4">
                  <div className="flex items-start justify-between mb-2">
                    <div className="flex-1">
                      <h4 className="font-medium text-gray-900 capitalize">{gap.area}</h4>
                      <p className="text-sm text-gray-600 mt-1">
                        Mentioned in {gap.evidenceCount} rejection{gap.evidenceCount > 1 ? 's' : ''}
                      </p>
                    </div>
                    <span className={`px-2 py-1 rounded text-xs font-medium ${
                      gap.priority === 'high' ? 'bg-red-100 text-red-700' :
                      gap.priority === 'medium' ? 'bg-yellow-100 text-yellow-700' :
                      'bg-gray-100 text-gray-700'
                    }`}>
                      {gap.priority}
                    </span>
                  </div>

                  <div className="flex items-center gap-2 mb-3">
                    <span className={`text-xs px-2 py-1 rounded ${
                      gap.remediationStatus === 'identified' ? 'bg-gray-100 text-gray-700' :
                      gap.remediationStatus === 'in_progress' ? 'bg-blue-100 text-blue-700' :
                      'bg-green-100 text-green-700'
                    }`}>
                      {gap.remediationStatus === 'identified' && 'Not Started'}
                      {gap.remediationStatus === 'in_progress' && 'In Progress'}
                      {gap.remediationStatus === 'completed' && 'Completed'}
                    </span>
                    <span className="text-xs text-gray-500">
                      Last seen: {new Date(gap.lastOccurrence).toLocaleDateString()}
                    </span>
                  </div>

                  {gap.remediationStatus === 'identified' && (
                    <button
                      onClick={() => startRemediationTracking(gap)}
                      className="w-full bg-blue-50 text-blue-700 py-2 rounded font-medium hover:bg-blue-100 transition-colors text-sm"
                    >
                      Start Addressing This Gap
                    </button>
                  )}

                  {gap.remediationStatus === 'in_progress' && (
                    <div className="bg-blue-50 p-3 rounded">
                      <p className="text-sm text-blue-800 font-medium">Active improvement plan</p>
                      <p className="text-xs text-blue-600 mt-1">
                        Started {gap.startedDate && new Date(gap.startedDate).toLocaleDateString()}
                      </p>
                    </div>
                  )}
                </div>
              ))}
            </div>
          )}
        </div>

        {userProfile.improvementTracking.length > 0 && (
          <div>
            <h3 className="font-semibold text-gray-900 mb-3 flex items-center gap-2">
              <TrendingUp className="w-5 h-5" />
              Active Improvements ({userProfile.improvementTracking.length})
            </h3>
            <div className="space-y-2">
              {userProfile.improvementTracking.map((track, idx) => (
                <div key={idx} className="bg-green-50 border border-green-200 rounded-lg p-3">
                  <div className="flex items-center justify-between">
                    <span className="font-medium text-green-900 capitalize">{track.gap}</span>
                    <span className="text-xs bg-green-100 text-green-700 px-2 py-1 rounded">
                      {track.status}
                    </span>
                  </div>
                  <p className="text-xs text-green-700 mt-1">
                    Started {new Date(track.startDate).toLocaleDateString()}
                  </p>
                </div>
              ))}
            </div>
          </div>
        )}
      </div>
    );
  };

  return (
    <div className="min-h-screen bg-gradient-to-br from-gray-900 via-gray-800 to-gray-900 p-6">
      <div className="max-w-6xl mx-auto">
        <div className="bg-gray-800 rounded-2xl shadow-2xl overflow-hidden border border-gray-700">
          <div className="bg-gradient-to-r from-blue-600 via-purple-600 to-pink-600 p-8 text-white relative overflow-hidden">
            <div className="absolute inset-0 bg-black opacity-10"></div>
            <div className="relative z-10">
              <div className="flex items-center gap-3 mb-3">
                <Brain className="w-10 h-10" />
                <h1 className="text-4xl font-bold">AI Career Rejection Engine</h1>
              </div>
              <p className="text-blue-100 text-lg">
                Transform rejections into actionable insights. Every "no" is data for your next "yes".
              </p>
            </div>
          </div>

          <div className="border-b border-gray-700 bg-gray-800">
            <div className="flex overflow-x-auto">
              {[
                { id: 'input', label: 'Add Rejection', icon: Brain },
                { id: 'insights', label: 'Insights', icon: BarChart3 },
                { id: 'actions', label: 'Action Plan', icon: Target },
                { id: 'profile', label: 'Progress', icon: TrendingUp }
              ].map(tab => {
                const Icon = tab.icon;
                return (
                  <button
                    key={tab.id}
                    onClick={() => setActiveTab(tab.id)}
                    className={`flex-1 px-6 py-4 flex items-center justify-center gap-2 font-medium transition-all ${
                      activeTab === tab.id
                        ? 'bg-gray-900 text-blue-400 border-b-2 border-blue-500'
                        : 'text-gray-400 hover:text-gray-200 hover:bg-gray-700'
                    }`}
                  >
                    <Icon className="w-5 h-5" />
                    {tab.label}
                  </button>
                );
              })}
            </div>
          </div>

          <div className="p-8 bg-gray-900">
            {activeTab === 'input' && <RejectionInput />}
            {activeTab === 'insights' && <InsightsDashboard />}
            {activeTab === 'actions' && <ActionPlan />}
            {activeTab === 'profile' && <ProfileTracker />}
          </div>
        </div>

        <div className="mt-6 text-center text-sm text-gray-500">
          <p className="flex items-center justify-center gap-2">
            <span className="text-2xl">💡</span>
            <span>Tip: The more rejections you track, the more accurate the pattern detection becomes</span>
          </p>
        </div>
      </div>
    </div>
  );
};

export default RejectionEngine;
