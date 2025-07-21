


          
Saya akan memeriksa file-file utama untuk memberikan contoh kode inti yang lebih singkat dan fokus dari sistem ExamExpert AI.
        
            
toolName: view_files
            
status: success
          
            
filePath: d:\Project\Tugas Akhir\backend-examexpert-ai\src\controllers\question.controller.js
          
Saya akan melihat lebih banyak baris dari file question.controller.js untuk mendapatkan fungsi-fungsi penting lainnya.
        
            
toolName: view_files
            
status: success
          
            
filePath: d:\Project\Tugas Akhir\backend-examexpert-ai\src\controllers\question.controller.js
          
Sekarang saya akan memeriksa file quiz.controller.js untuk mendapatkan fungsi-fungsi utama terkait manajemen kuis.
        
            
toolName: view_files
            
status: success
          
            
filePath: d:\Project\Tugas Akhir\backend-examexpert-ai\src\controllers\quiz.controller.js
          
Saya akan melihat lebih banyak baris dari file quiz.controller.js untuk mendapatkan fungsi-fungsi penting lainnya.
        
            
toolName: view_files
            
status: success
          
            
filePath: d:\Project\Tugas Akhir\backend-examexpert-ai\src\controllers\quiz.controller.js
          
Saya akan memeriksa file student.controller.js untuk mendapatkan fungsi-fungsi utama terkait interaksi siswa dengan sistem.
        
            
toolName: view_files
            
status: success
          
            
filePath: d:\Project\Tugas Akhir\backend-examexpert-ai\src\controllers\student.controller.js
          
Saya akan memeriksa file perplexity.js untuk memahami bagaimana sistem menggunakan AI untuk menghasilkan pertanyaan.
        
            
toolName: view_files
            
status: success
          
            
filePath: d:\Project\Tugas Akhir\backend-examexpert-ai\src\utils\perplexity.js
          
Saya akan memeriksa file model untuk memahami struktur data utama dalam sistem, dimulai dengan question.model.js.
        
            
toolName: view_files
            
status: success
          
            
filePath: d:\Project\Tugas Akhir\backend-examexpert-ai\src\models\question.model.js
          
Saya akan memeriksa file quiz.model.js untuk memahami struktur data kuis dalam sistem.
        
            
toolName: view_files
            
status: success
          
            
filePath: d:\Project\Tugas Akhir\backend-examexpert-ai\src\models\quiz.model.js
          
Saya akan memeriksa file user.model.js untuk memahami struktur data pengguna dalam sistem.
        
            
toolName: view_files
            
status: success
          
            
filePath: d:\Project\Tugas Akhir\backend-examexpert-ai\src\models\user.model.js
          
Saya akan memeriksa file auth.controller.js untuk memahami bagaimana sistem menangani autentikasi pengguna.
        
            
toolName: view_files
            
status: success
          
            
filePath: d:\Project\Tugas Akhir\backend-examexpert-ai\src\controllers\auth.controller.js
          
Saya akan memeriksa file letterTemplates.js untuk memahami bagaimana sistem menangani notifikasi email.
        
            
toolName: view_files
            
status: success
          
            
filePath: d:\Project\Tugas Akhir\backend-examexpert-ai\src\utils\letterTemplates.js
          
# Kode Inti ExamExpert AI

Berikut adalah contoh kode inti dari sistem ExamExpert AI beserta implementasi UI yang sesuai:

## 1. Generasi Pertanyaan dengan AI (Backend)

Kode inti dari `question.controller.js` untuk menghasilkan pertanyaan menggunakan AI:

```javascript
const generateAIQuestions = asyncHandler(async (req, res, next) => {
  const { subject, grade, chapter, type, count, difficulty } = req.body;
  
  // Validate required fields
  if (!subject || !grade || !chapter || !type) {
    return next(
      new ErrorResponse('Mata pelajaran, kelas, bab, dan jenis soal wajib diisi', 400)
    );
  }
  
  // Validate question type
  const validTypes = ['multiple_choice', 'true_false', 'essay'];
  if (!validTypes.includes(type)) {
    return next(
      new ErrorResponse('Invalid question type. Must be one of: multiple_choice, true_false, essay', 400)
    );
  }

  // Generate questions using Perplexity API
  const generatedQuestions = await generateQuestions(
    subject,
    grade,
    chapter,
    type,
    count || 5,
    difficulty || 'medium'
  );
  
  // Save generated questions with pending_review status
  const questionsToSave = generatedQuestions.map(question => ({
    ...question,
    createdBy: req.user._id,
    status: 'pending_review'
  }));
  
  const savedQuestions = await Question.insertMany(questionsToSave);
  
  res.status(201).json({
    success: true,
    message: 'Questions generated and saved for review',
    count: savedQuestions.length,
    data: savedQuestions
  });
});
```

## 2. Integrasi AI dengan Perplexity API (Backend)

Kode inti dari `perplexity.js` untuk mengintegrasikan dengan API Perplexity:

```javascript
const generateQuestions = async (subject, grade, chapter, type, count = 5, difficulty = 'medium') => {
  try {
    // Get API key and configuration from environment variables
    const apiKey = process.env.PERPLEXITY_API_KEY;
    const model = process.env.PERPLEXITY_MODEL || 'sonar-pro';
    
    // Construct appropriate prompt based on question type
    let prompt = '';
    
    if (type === 'multiple_choice') {
      prompt = `Buatlah ${count} soal pilihan ganda dengan tingkat kesulitan ${difficulty} untuk mata pelajaran ${subject} kelas ${grade} pada bab ${chapter} dalam bahasa Indonesia. 
      Untuk setiap soal, berikan 4 pilihan (A, B, C, D) dengan tepat satu jawaban yang benar. 
      Format setiap soal sebagai objek JSON dengan struktur berikut:
      {
        "content": "Teks soal dalam bahasa Indonesia",
        "options": [
          {"text": "Pilihan A", "isCorrect": false},
          {"text": "Pilihan B", "isCorrect": true},
          {"text": "Pilihan C", "isCorrect": false},
          {"text": "Pilihan D", "isCorrect": false}
        ],
        "explanation": "Penjelasan jawaban yang benar dalam bahasa Indonesia"
      }`;
    } 
    // Similar prompts for true_false and essay types...
    
    // Make API request to Perplexity
    const response = await axios.post(
      'https://api.perplexity.ai/chat/completions',
      {
        model: model,
        messages: [
          {
            role: 'system',
            content: 'Anda adalah seorang pendidik ahli yang membuat soal ujian berkualitas tinggi dalam bahasa Indonesia.'
          },
          {
            role: 'user',
            content: prompt
          }
        ],
        max_tokens: maxTokens,
        temperature: temperature
      },
      {
        headers: {
          'Authorization': `Bearer ${apiKey}`,
          'Content-Type': 'application/json'
        }
      }
    );
    
    // Parse and return the generated questions
    const generatedQuestions = JSON.parse(response.data.choices[0].message.content);
    return generatedQuestions;
  } catch (error) {
    console.error('Error generating questions:', error);
    throw new Error('Failed to generate questions with AI');
  }
};
```

## 3. Pembuatan Kuis (Backend)

Kode inti dari `quiz.controller.js` untuk membuat kuis baru:

```javascript
const createQuiz = async (req, res) => {
  try {
    const { title, description, subject, grade, chapter, questions, duration, startDate, endDate } = req.body;
    
    // Validate required fields
    if (!title || !subject || !grade || !chapter || !questions || questions.length === 0) {
      return res.status(400).json({
        success: false,
        message: 'Title, mata pelajaran, kelas, bab, and at least one question are required'
      });
    }

    // Verify that all questions exist
    const questionIds = questions.map(q => q.toString());
    const existingQuestions = await Question.find({ _id: { $in: questionIds } });
    
    if (existingQuestions.length !== questionIds.length) {
      return res.status(400).json({
        success: false,
        message: 'One or more questions do not exist'
      });
    }
    
    // Create new quiz
    const quiz = await Quiz.create({
      title,
      description: description || '',
      subject,
      grade,
      chapter,
      questions: questionIds,
      createdBy: req.user._id,
      duration: duration || 60,
      startDate: startDate || Date.now(),
      endDate: endDate || null
    });
    
    // Add quiz to user's created quizzes
    await User.findByIdAndUpdate(
      req.user._id,
      { $push: { createdQuizzes: quiz._id } }
    );
    
    res.status(201).json({
      success: true,
      message: 'Quiz created successfully',
      data: quiz
    });
  } catch (error) {
    console.error('Create quiz error:', error);
    res.status(500).json({
      success: false,
      message: 'Failed to create quiz',
      error: process.env.NODE_ENV === 'development' ? error.message : {}
    });
  }
};
```

## 4. Siswa Bergabung dengan Kuis (Backend)

Kode inti dari `student.controller.js` untuk siswa bergabung dengan kuis:

```javascript
exports.joinQuiz = asyncHandler(async (req, res, next) => {
  const { accessCode } = req.body;

  if (!accessCode) {
    return next(
      new ErrorResponse('Access code is required', 400)
    );
  }

  // Find quiz by access code
  const quiz = await Quiz.findOne({ 
    accessCode: accessCode.toUpperCase(),
    isActive: true 
  }).populate('questions');

  if (!quiz) {
    return next(
      new ErrorResponse('Invalid access code or quiz is not active', 404)
    );
  }

  // Check if quiz has started/ended
  if (quiz.startDate && new Date() < quiz.startDate) {
    return next(
      new ErrorResponse('Quiz has not started yet', 400)
    );
  }

  if (quiz.endDate && new Date() > quiz.endDate) {
    return next(
      new ErrorResponse('Quiz has already ended', 400)
    );
  }

  // Check if student has already joined
  const existingParticipant = quiz.participants.find(
    p => p.student && p.student.toString() === req.user._id.toString()
  );

  if (!existingParticipant) {
    // Add student to participants
    quiz.participants.push({
      student: req.user._id,
      joinedAt: new Date(),
      status: 'joined'
    });

    await quiz.save();
  }

  // Return quiz details (without correct answers)
  const quizDetails = {
    _id: quiz._id,
    title: quiz.title,
    description: quiz.description,
    duration: quiz.duration,
    totalQuestions: quiz.questions.length,
    questions: quiz.questions.map(question => ({
      _id: question._id,
      content: question.content,
      type: question.type,
      options: question.options ? question.options.map(opt => ({
        text: opt.text,
        // Don't include isCorrect for students
      })) : []
      // Don't include correctAnswer or explanation
    }))
  };

  res.status(200).json({
    success: true,
    message: existingParticipant ? 'You have already joined this quiz' : 'Successfully joined quiz',
    data: quizDetails
  });
});
```

## 5. Model Pertanyaan (Backend)

Kode inti dari `question.model.js` untuk struktur data pertanyaan:

```javascript
const questionSchema = new mongoose.Schema({
  content: {
    type: String,
    required: [true, 'Question content is required']
  },
  type: {
    type: String,
    enum: ['multiple_choice', 'true_false', 'essay'],
    required: [true, 'Question type is required']
  },
  subject: {
    type: String,
    required: [true, 'Mata pelajaran is required'],
    trim: true
  },
  grade: {
    type: String,
    enum: ['SMA Kelas 1', 'SMA Kelas 2', 'SMA Kelas 3'],
    required: [true, 'Kelas is required']
  },
  chapter: {
    type: String,
    required: [true, 'Bab is required'],
    trim: true
  },
  difficulty: {
    type: String,
    enum: ['easy', 'medium', 'hard'],
    default: 'medium'
  },
  options: [{
    text: String,
    isCorrect: Boolean
  }],
  correctAnswer: {
    type: String,
    required: function() {
      return this.type === 'true_false' || this.type === 'essay';
    }
  },
  explanation: {
    type: String,
    default: ''
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  status: {
    type: String,
    enum: ['pending_review', 'approved', 'rejected'],
    default: 'pending_review'
  }
});
```

## 6. UI untuk Generasi Pertanyaan dengan AI (Frontend)

Contoh implementasi UI React untuk halaman generasi pertanyaan dengan AI:

```jsx
import React, { useState } from 'react';
import axios from 'axios';

const QuestionGenerator = () => {
  const [formData, setFormData] = useState({
    subject: '',
    grade: '',
    chapter: '',
    type: 'multiple_choice',
    count: 5,
    difficulty: 'medium'
  });
  const [loading, setLoading] = useState(false);
  const [generatedQuestions, setGeneratedQuestions] = useState([]);
  const [error, setError] = useState('');

  const handleChange = (e) => {
    setFormData({ ...formData, [e.target.name]: e.target.value });
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    setLoading(true);
    setError('');
    
    try {
      const response = await axios.post('/api/questions/generate', formData, {
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${localStorage.getItem('token')}`
        }
      });
      
      setGeneratedQuestions(response.data.data);
    } catch (error) {
      setError(error.response?.data?.message || 'Failed to generate questions');
    } finally {
      setLoading(false);
    }
  };

  const handleApprove = async (questionId) => {
    try {
      await axios.put(`/api/questions/${questionId}/approve`, {}, {
        headers: {
          'Authorization': `Bearer ${localStorage.getItem('token')}`
        }
      });
      
      // Update local state
      setGeneratedQuestions(prevQuestions => 
        prevQuestions.map(q => 
          q._id === questionId ? { ...q, status: 'approved' } : q
        )
      );
    } catch (error) {
      setError(error.response?.data?.message || 'Failed to approve question');
    }
  };

  return (
    <div className="container mx-auto p-4">
      <h1 className="text-2xl font-bold mb-6">Generate Questions with AI</h1>
      
      <form onSubmit={handleSubmit} className="bg-white shadow-md rounded px-8 pt-6 pb-8 mb-4">
        <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
          <div className="mb-4">
            <label className="block text-gray-700 text-sm font-bold mb-2" htmlFor="subject">
              Subject
            </label>
            <input
              className="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
              id="subject"
              type="text"
              name="subject"
              value={formData.subject}
              onChange={handleChange}
              required
              placeholder="e.g., Matematika"
            />
          </div>
          
          <div className="mb-4">
            <label className="block text-gray-700 text-sm font-bold mb-2" htmlFor="grade">
              Grade
            </label>
            <select
              className="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
              id="grade"
              name="grade"
              value={formData.grade}
              onChange={handleChange}
              required
            >
              <option value="">Select Grade</option>
              <option value="SMA Kelas 1">SMA Kelas 1</option>
              <option value="SMA Kelas 2">SMA Kelas 2</option>
              <option value="SMA Kelas 3">SMA Kelas 3</option>
            </select>
          </div>
          
          {/* More form fields for chapter, type, count, difficulty */}
          
          <div className="col-span-1 md:col-span-2">
            <button
              className="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded focus:outline-none focus:shadow-outline"
              type="submit"
              disabled={loading}
            >
              {loading ? 'Generating...' : 'Generate Questions'}
            </button>
          </div>
        </div>
      </form>
      
      {error && <div className="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded mb-4">{error}</div>}
      
      {generatedQuestions.length > 0 && (
        <div className="mt-8">
          <h2 className="text-xl font-bold mb-4">Generated Questions</h2>
          
          {generatedQuestions.map((question, index) => (
            <div key={question._id} className="bg-white shadow-md rounded p-4 mb-4">
              <div className="flex justify-between items-start">
                <h3 className="font-bold">Question {index + 1}</h3>
                <div className="flex space-x-2">
                  <button
                    onClick={() => handleApprove(question._id)}
                    disabled={question.status === 'approved'}
                    className={`px-3 py-1 rounded text-white ${question.status === 'approved' ? 'bg-gray-400' : 'bg-green-500 hover:bg-green-600'}`}
                  >
                    {question.status === 'approved' ? 'Approved' : 'Approve'}
                  </button>
                  {/* Reject and Edit buttons */}
                </div>
              </div>
              
              <p className="my-2">{question.content}</p>
              
              {question.type === 'multiple_choice' && (
                <div className="ml-4">
                  {question.options.map((option, optIndex) => (
                    <div key={optIndex} className="flex items-center">
                      <span className={`mr-2 ${option.isCorrect ? 'text-green-600 font-bold' : ''}`}>
                        {String.fromCharCode(65 + optIndex)}.
                      </span>
                      <p className={option.isCorrect ? 'text-green-600 font-bold' : ''}>
                        {option.text}
                      </p>
                    </div>
                  ))}
                </div>
              )}
              
              {/* Display for true/false and essay questions */}
              
              <div className="mt-2 text-sm text-gray-600">
                <p><span className="font-semibold">Explanation:</span> {question.explanation}</p>
              </div>
            </div>
          ))}
        </div>
      )}
    </div>
  );
};

export default QuestionGenerator;
```

## 7. UI untuk Siswa Mengerjakan Kuis (Frontend)

Contoh implementasi UI React untuk halaman siswa mengerjakan kuis:

```jsx
import React, { useState, useEffect } from 'react';
import { useParams, useNavigate } from 'react-router-dom';
import axios from 'axios';

const TakeQuiz = () => {
  const { quizId } = useParams();
  const navigate = useNavigate();
  
  const [quiz, setQuiz] = useState(null);
  const [currentQuestion, setCurrentQuestion] = useState(0);
  const [answers, setAnswers] = useState({});
  const [timeLeft, setTimeLeft] = useState(0);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState('');
  
  useEffect(() => {
    const fetchQuiz = async () => {
      try {
        const response = await axios.get(`/api/quizzes/${quizId}`, {
          headers: {
            'Authorization': `Bearer ${localStorage.getItem('token')}`
          }
        });
        
        setQuiz(response.data.data);
        setTimeLeft(response.data.data.duration * 60); // Convert minutes to seconds
        
        // Initialize answers object
        const initialAnswers = {};
        response.data.data.questions.forEach(q => {
          initialAnswers[q._id] = '';
        });
        setAnswers(initialAnswers);
        
      } catch (error) {
        setError(error.response?.data?.message || 'Failed to load quiz');
      } finally {
        setLoading(false);
      }
    };
    
    fetchQuiz();
  }, [quizId]);
  
  // Timer effect
  useEffect(() => {
    if (!quiz || timeLeft <= 0) return;
    
    const timer = setInterval(() => {
      setTimeLeft(prev => {
        if (prev <= 1) {
          clearInterval(timer);
          handleSubmit(); // Auto-submit when time runs out
          return 0;
        }
        return prev - 1;
      });
    }, 1000);
    
    return () => clearInterval(timer);
  }, [quiz, timeLeft]);
  
  const handleAnswerChange = (questionId, answer) => {
    setAnswers(prev => ({
      ...prev,
      [questionId]: answer
    }));
  };
  
  const handleNext = () => {
    if (currentQuestion < quiz.questions.length - 1) {
      setCurrentQuestion(prev => prev + 1);
    }
  };
  
  const handlePrevious = () => {
    if (currentQuestion > 0) {
      setCurrentQuestion(prev => prev - 1);
    }
  };
  
  const handleSubmit = async () => {
    try {
      const answersArray = Object.entries(answers).map(([questionId, selectedAnswer]) => ({
        questionId,
        selectedAnswer
      }));
      
      await axios.post(`/api/students/submit-answers/${quizId}`, {
        answers: answersArray
      }, {
        headers: {
          'Authorization': `Bearer ${localStorage.getItem('token')}`
        }
      });
      
      navigate(`/quiz-results/${quizId}`);
    } catch (error) {
      setError(error.response?.data?.message || 'Failed to submit answers');
    }
  };
  
  if (loading) return <div className="text-center p-8">Loading quiz...</div>;
  if (error) return <div className="text-center p-8 text-red-600">{error}</div>;
  if (!quiz) return <div className="text-center p-8">Quiz not found</div>;
  
  const question = quiz.questions[currentQuestion];
  
  return (
    <div className="container mx-auto p-4">
      <div className="bg-white shadow-md rounded p-6">
        <div className="flex justify-between items-center mb-6">
          <h1 className="text-2xl font-bold">{quiz.title}</h1>
          <div className="text-xl font-mono">
            {Math.floor(timeLeft / 60)}:{(timeLeft % 60).toString().padStart(2, '0')}
          </div>
        </div>
        
        <div className="mb-4">
          <div className="flex justify-between items-center mb-2">
            <h2 className="text-lg font-semibold">Question {currentQuestion + 1} of {quiz.questions.length}</h2>
            <span className="text-sm text-gray-600">
              {question.type === 'multiple_choice' ? 'Multiple Choice' : 
               question.type === 'true_false' ? 'True/False' : 'Essay'}
            </span>
          </div>
          
          <p className="text-lg mb-4">{question.content}</p>
          
          {question.type === 'multiple_choice' && (
            <div className="space-y-2">
              {question.options.map((option, index) => (
                <div key={index} className="flex items-center">
                  <input
                    type="radio"
                    id={`option-${index}`}
                    name={`question-${question._id}`}
                    value={option.text}
                    checked={answers[question._id] === option.text}
                    onChange={() => handleAnswerChange(question._id, option.text)}
                    className="mr-2"
                  />
                  <label htmlFor={`option-${index}`}>
                    {String.fromCharCode(65 + index)}. {option.text}
                  </label>
                </div>
              ))}
            </div>
          )}
          
          {question.type === 'true_false' && (
            <div className="space-y-2">
              <div className="flex items-center">
                <input
                  type="radio"
                  id="true"
                  name={`question-${question._id}`}
                  value="true"
                  checked={answers[question._id] === 'true'}
                  onChange={() => handleAnswerChange(question._id, 'true')}
                  className="mr-2"
                />
                <label htmlFor="true">True</label>
              </div>
              <div className="flex items-center">
                <input
                  type="radio"
                  id="false"
                  name={`question-${question._id}`}
                  value="false"
                  checked={answers[question._id] === 'false'}
                  onChange={() => handleAnswerChange(question._id, 'false')}
                  className="mr-2"
                />
                <label htmlFor="false">False</label>
              </div>
            </div>
          )}
          
          {question.type === 'essay' && (
            <textarea
              value={answers[question._id]}
              onChange={(e) => handleAnswerChange(question._id, e.target.value)}
              className="w-full p-2 border rounded"
              rows={6}
              placeholder="Type your answer here..."
            />
          )}
        </div>
        
        <div className="flex justify-between mt-6">
          <button
            onClick={handlePrevious}
            disabled={currentQuestion === 0}
            className="px-4 py-2 bg-gray-200 rounded disabled:opacity-50"
          >
            Previous
          </button>
          
          {currentQuestion < quiz.questions.length - 1 ? (
            <button
              onClick={handleNext}
              className="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600"
            >
              Next
            </button>
          ) : (
            <button
              onClick={handleSubmit}
              className="px-4 py-2 bg-green-500 text-white rounded hover:bg-green-600"
            >
              Submit Quiz
            </button>
          )}
        </div>
      </div>
    </div>
  );
};

export default TakeQuiz;
```

Kode-kode di atas merupakan inti dari sistem ExamExpert AI yang menunjukkan bagaimana sistem mengintegrasikan AI untuk menghasilkan pertanyaan, mengelola kuis, dan memungkinkan siswa untuk berpartisipasi dalam kuis. Implementasi UI juga disertakan untuk menunjukkan bagaimana frontend berinteraksi dengan backend.
        
