


          
# Kode Inti ExamExpert AI dengan Penjelasan Detail

Berikut adalah contoh kode inti dari sistem ExamExpert AI dengan penjelasan detail untuk setiap fungsi:

## 1. Generasi Pertanyaan dengan AI (Backend)

Kode inti dari `question.controller.js` untuk menghasilkan pertanyaan menggunakan AI:

```javascript
const generateAIQuestions = asyncHandler(async (req, res, next) => {
  // Ekstrak data dari request body
  const { subject, grade, chapter, type, count, difficulty } = req.body;
  
  // Validasi field yang wajib diisi
  if (!subject || !grade || !chapter || !type) {
    return next(
      new ErrorResponse('Mata pelajaran, kelas, bab, dan jenis soal wajib diisi', 400)
    );
  }
  
  // Validasi tipe pertanyaan harus sesuai dengan yang didukung sistem
  const validTypes = ['multiple_choice', 'true_false', 'essay'];
  if (!validTypes.includes(type)) {
    return next(
      new ErrorResponse('Invalid question type. Must be one of: multiple_choice, true_false, essay', 400)
    );
  }

  // Validasi kelas harus sesuai dengan yang didukung sistem
  const validGrades = ['SMA Kelas 1', 'SMA Kelas 2', 'SMA Kelas 3'];
  if (!validGrades.includes(grade)) {
    return next(
      new ErrorResponse('Invalid grade. Must be one of: SMA Kelas 1, SMA Kelas 2, SMA Kelas 3', 400)
    );
  }
  
  // Panggil fungsi generateQuestions dari utilitas perplexity.js untuk menghasilkan pertanyaan dengan AI
  const generatedQuestions = await generateQuestions(
    subject,
    grade,
    chapter,
    type,
    count || 5, // Default 5 pertanyaan jika tidak ditentukan
    difficulty || 'medium' // Default tingkat kesulitan medium jika tidak ditentukan
  );
  
  // Tambahkan metadata ke pertanyaan yang dihasilkan (pembuat dan status)
  const questionsToSave = generatedQuestions.map(question => ({
    ...question,
    createdBy: req.user._id, // ID pengguna yang membuat pertanyaan
    status: 'pending_review' // Status awal adalah menunggu review
  }));
  
  // Simpan pertanyaan ke database
  const savedQuestions = await Question.insertMany(questionsToSave);
  
  // Kirim respons sukses dengan data pertanyaan yang disimpan
  res.status(201).json({
    success: true,
    message: 'Questions generated and saved for review',
    count: savedQuestions.length,
    data: savedQuestions
  });
});
```

**Penjelasan Fungsi:**
- Fungsi ini menerima parameter mata pelajaran, kelas, bab, tipe pertanyaan, jumlah, dan tingkat kesulitan
- Melakukan validasi input untuk memastikan semua field yang diperlukan telah diisi
- Memanggil API Perplexity untuk menghasilkan pertanyaan berdasarkan parameter yang diberikan
- Menyimpan pertanyaan yang dihasilkan ke database dengan status 'pending_review'
- Mengembalikan respons dengan data pertanyaan yang berhasil disimpan

## 2. Integrasi AI dengan Perplexity API (Backend)

Kode inti dari `perplexity.js` untuk mengintegrasikan dengan API Perplexity:

```javascript
const generateQuestions = async (subject, grade, chapter, type, count = 5, difficulty = 'medium') => {
  try {
    // Ambil API key dan konfigurasi dari environment variables
    const apiKey = process.env.PERPLEXITY_API_KEY;
    const model = process.env.PERPLEXITY_MODEL || 'sonar-pro';
    const maxTokens = parseInt(process.env.PERPLEXITY_MAX_TOKENS) || 4000;
    const temperature = parseFloat(process.env.PERPLEXITY_TEMPERATURE) || 0.7;
    const topP = parseFloat(process.env.PERPLEXITY_TOP_P) || 0.9;
    
    // Validasi API key
    if (!apiKey) {
      throw new Error('Perplexity API key is missing');
    }
    
    // Buat prompt yang sesuai berdasarkan tipe pertanyaan
    let prompt = '';
    
    if (type === 'multiple_choice') {
      // Prompt untuk pertanyaan pilihan ganda
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
    // Prompt untuk tipe pertanyaan lainnya (true_false dan essay) dihilangkan untuk singkatnya
    
    // Kirim permintaan ke API Perplexity
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
        temperature: temperature,
        top_p: topP
      },
      {
        headers: {
          'Authorization': `Bearer ${apiKey}`,
          'Content-Type': 'application/json'
        }
      }
    );
    
    // Parse dan kembalikan pertanyaan yang dihasilkan
    const generatedQuestions = JSON.parse(response.data.choices[0].message.content);
    return generatedQuestions;
  } catch (error) {
    console.error('Error generating questions:', error);
    throw new Error('Failed to generate questions with AI');
  }
};
```

**Penjelasan Fungsi:**
- Fungsi ini mengambil konfigurasi API dari environment variables untuk keamanan
- Membuat prompt yang berbeda berdasarkan tipe pertanyaan (multiple_choice, true_false, atau essay)
- Mengirim permintaan ke API Perplexity dengan prompt yang telah dibuat
- Memproses respons dari API dan mengembalikan pertanyaan yang dihasilkan dalam format JSON
- Menangani error yang mungkin terjadi selama proses komunikasi dengan API

## 3. Pembuatan Kuis (Backend)

Kode inti dari `quiz.controller.js` untuk membuat kuis baru:

```javascript
const createQuiz = async (req, res) => {
  try {
    // Ekstrak data dari request body
    const { title, description, subject, grade, chapter, questions, duration, startDate, endDate } = req.body;
    
    // Validasi field yang wajib diisi
    if (!title || !subject || !grade || !chapter || !questions || questions.length === 0) {
      return res.status(400).json({
        success: false,
        message: 'Title, mata pelajaran, kelas, bab, and at least one question are required'
      });
    }

    // Validasi kelas
    const validGrades = ['SMA Kelas 1', 'SMA Kelas 2', 'SMA Kelas 3'];
    if (!validGrades.includes(grade)) {
      return res.status(400).json({
        success: false,
        message: 'Invalid grade. Must be one of: SMA Kelas 1, SMA Kelas 2, SMA Kelas 3'
      });
    }
    
    // Verifikasi bahwa semua pertanyaan yang dipilih ada di database
    const questionIds = questions.map(q => q.toString());
    const existingQuestions = await Question.find({ _id: { $in: questionIds } });
    
    if (existingQuestions.length !== questionIds.length) {
      return res.status(400).json({
        success: false,
        message: 'One or more questions do not exist'
      });
    }
    
    // Buat kuis baru
    const quiz = await Quiz.create({
      title,
      description: description || '',
      subject,
      grade,
      chapter,
      questions: questionIds,
      createdBy: req.user._id, // ID pengguna yang membuat kuis
      duration: duration || 60, // Default durasi 60 menit
      startDate: startDate || Date.now(), // Default mulai sekarang
      endDate: endDate || null // Default tidak ada tanggal berakhir
    });
    
    // Tambahkan kuis ke daftar kuis yang dibuat oleh pengguna
    await User.findByIdAndUpdate(
      req.user._id,
      { $push: { createdQuizzes: quiz._id } }
    );
    
    // Kirim email konfirmasi pembuatan kuis (opsional)
    try {
      await sendQuizCreationConfirmationLetter({
        teacherName: req.user.fullName || req.user.name,
        teacherEmail: req.user.email,
        quizTitle: quiz.title,
        accessCode: quiz.accessCode,
        quizDescription: quiz.description,
        duration: quiz.duration,
        questionCount: quiz.questions.length,
        endDate: quiz.endDate
      });
    } catch (emailError) {
      // Jangan gagalkan pembuatan kuis jika email gagal terkirim
      console.warn('Failed to send quiz creation confirmation email:', emailError.message);
    }
    
    // Kirim respons sukses dengan data kuis yang dibuat
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

**Penjelasan Fungsi:**
- Fungsi ini menerima parameter judul, deskripsi, mata pelajaran, kelas, bab, daftar pertanyaan, durasi, tanggal mulai, dan tanggal berakhir
- Melakukan validasi input untuk memastikan semua field yang diperlukan telah diisi
- Memverifikasi bahwa semua pertanyaan yang dipilih ada di database
- Membuat kuis baru dengan parameter yang diberikan dan menghasilkan kode akses unik secara otomatis
- Menambahkan kuis ke daftar kuis yang dibuat oleh pengguna
- Mengirim email konfirmasi pembuatan kuis (opsional)
- Mengembalikan respons dengan data kuis yang berhasil dibuat

## 4. Siswa Bergabung dengan Kuis (Backend)

Kode inti dari `student.controller.js` untuk siswa bergabung dengan kuis:

```javascript
exports.joinQuiz = asyncHandler(async (req, res, next) => {
  // Ekstrak kode akses dari request body
  const { accessCode } = req.body;

  // Validasi kode akses
  if (!accessCode) {
    return next(
      new ErrorResponse('Access code is required', 400)
    );
  }

  // Cari kuis berdasarkan kode akses
  const quiz = await Quiz.findOne({ 
    accessCode: accessCode.toUpperCase(), // Konversi ke huruf besar untuk konsistensi
    isActive: true // Hanya kuis yang aktif
  }).populate('questions'); // Ambil detail pertanyaan

  // Validasi kuis ditemukan
  if (!quiz) {
    return next(
      new ErrorResponse('Invalid access code or quiz is not active', 404)
    );
  }

  // Periksa apakah kuis sudah dimulai
  if (quiz.startDate && new Date() < quiz.startDate) {
    return next(
      new ErrorResponse('Quiz has not started yet', 400)
    );
  }

  // Periksa apakah kuis sudah berakhir
  if (quiz.endDate && new Date() > quiz.endDate) {
    return next(
      new ErrorResponse('Quiz has already ended', 400)
    );
  }

  // Periksa apakah siswa sudah bergabung sebelumnya
  const existingParticipant = quiz.participants.find(
    p => p.student && p.student.toString() === req.user._id.toString()
  );

  // Jika siswa sudah bergabung, kembalikan detail kuis
  if (existingParticipant) {
    const quizDetails = {
      _id: quiz._id,
      title: quiz.title,
      description: quiz.description,
      duration: quiz.duration,
      totalQuestions: quiz.questions.length,
      startDate: quiz.startDate,
      endDate: quiz.endDate,
      accessCode: quiz.accessCode,
      alreadyJoined: true,
      participantStatus: existingParticipant.status,
      questions: quiz.questions.map(question => ({
        _id: question._id,
        content: question.content,
        type: question.type,
        options: question.options ? question.options.map(opt => ({
          text: opt.text,
          // Jangan sertakan isCorrect untuk siswa
        })) : []
        // Jangan sertakan correctAnswer atau explanation
      }))
    };

    return res.status(200).json({
      success: true,
      message: 'You have already joined this quiz',
      data: quizDetails
    });
  }

  // Tambahkan siswa ke daftar peserta
  quiz.participants.push({
    student: req.user._id,
    joinedAt: new Date(),
    status: 'joined'
  });

  // Simpan perubahan
  await quiz.save();
  
  // Siapkan detail kuis untuk dikirim ke siswa (tanpa jawaban benar)
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
        // Jangan sertakan isCorrect untuk siswa
      })) : []
      // Jangan sertakan correctAnswer atau explanation
    }))
  };

  // Kirim respons sukses dengan detail kuis
  res.status(200).json({
    success: true,
    message: 'Successfully joined quiz',
    data: quizDetails
  });
});
```

**Penjelasan Fungsi:**
- Fungsi ini menerima parameter kode akses kuis
- Memvalidasi kode akses dan mencari kuis yang sesuai di database
- Memeriksa apakah kuis aktif, sudah dimulai, dan belum berakhir
- Memeriksa apakah siswa sudah bergabung sebelumnya
- Jika siswa belum bergabung, menambahkan siswa ke daftar peserta kuis
- Mengembalikan detail kuis tanpa jawaban benar untuk menjaga integritas kuis

## 5. Model Pertanyaan (Backend)

Kode inti dari `question.model.js` untuk struktur data pertanyaan:

```javascript
const questionSchema = new mongoose.Schema({
  // Konten pertanyaan
  content: {
    type: String,
    required: [true, 'Question content is required']
  },
  // Tipe pertanyaan (pilihan ganda, benar/salah, essay)
  type: {
    type: String,
    enum: ['multiple_choice', 'true_false', 'essay'],
    required: [true, 'Question type is required']
  },
  // Mata pelajaran
  subject: {
    type: String,
    required: [true, 'Mata pelajaran is required'],
    trim: true
  },
  // Kelas (SMA Kelas 1, 2, atau 3)
  grade: {
    type: String,
    enum: ['SMA Kelas 1', 'SMA Kelas 2', 'SMA Kelas 3'],
    required: [true, 'Kelas is required']
  },
  // Bab dalam mata pelajaran
  chapter: {
    type: String,
    required: [true, 'Bab is required'],
    trim: true
  },
  // Tingkat kesulitan
  difficulty: {
    type: String,
    enum: ['easy', 'medium', 'hard'],
    default: 'medium'
  },
  // Pilihan jawaban untuk pertanyaan pilihan ganda
  options: [{
    text: String, // Teks pilihan
    isCorrect: Boolean // Apakah pilihan ini jawaban yang benar
  }],
  // Jawaban benar untuk pertanyaan benar/salah dan essay
  correctAnswer: {
    type: String,
    // Wajib diisi untuk pertanyaan benar/salah dan essay
    required: function() {
      return this.type === 'true_false' || this.type === 'essay';
    }
  },
  // Penjelasan jawaban
  explanation: {
    type: String,
    default: ''
  },
  // Pengguna yang membuat pertanyaan
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  // Status pertanyaan (menunggu review, disetujui, ditolak)
  status: {
    type: String,
    enum: ['pending_review', 'approved', 'rejected'],
    default: 'pending_review'
  },
  // Apakah pertanyaan sudah disetujui
  isApproved: {
    type: Boolean,
    default: false
  },
  // Pengguna yang menyetujui pertanyaan
  approvedBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  // Waktu persetujuan
  approvedAt: {
    type: Date
  },
  // Pengguna yang menolak pertanyaan
  rejectedBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  // Waktu penolakan
  rejectedAt: {
    type: Date
  },
  // Alasan penolakan
  rejectionReason: {
    type: String
  },
  // Apakah pertanyaan dihasilkan oleh AI
  aiGenerated: {
    type: Boolean,
    default: false
  }
}, {
  timestamps: true // Tambahkan createdAt dan updatedAt secara otomatis
});

// Indeks untuk pencarian efisien
questionSchema.index({ subject: 1, grade: 1, chapter: 1, type: 1, difficulty: 1 });
```

**Penjelasan Model:**
- Model ini mendefinisikan struktur data untuk pertanyaan dalam sistem
- Mencakup berbagai tipe pertanyaan (pilihan ganda, benar/salah, essay)
- Menyimpan metadata seperti mata pelajaran, kelas, bab, dan tingkat kesulitan
- Melacak status persetujuan dan penolakan pertanyaan
- Menyimpan informasi tentang pembuat pertanyaan dan reviewer
- Menggunakan indeks untuk meningkatkan performa pencarian

## 6. UI untuk Generasi Pertanyaan dengan AI (Frontend)

Contoh implementasi UI React untuk halaman generasi pertanyaan dengan AI:

```jsx
import React, { useState } from 'react';
import axios from 'axios';

const QuestionGenerator = () => {
  // State untuk form data
  const [formData, setFormData] = useState({
    subject: '',
    grade: '',
    chapter: '',
    type: 'multiple_choice',
    count: 5,
    difficulty: 'medium'
  });
  // State untuk loading, pertanyaan yang dihasilkan, dan error
  const [loading, setLoading] = useState(false);
  const [generatedQuestions, setGeneratedQuestions] = useState([]);
  const [error, setError] = useState('');

  // Handler untuk perubahan input form
  const handleChange = (e) => {
    setFormData({ ...formData, [e.target.name]: e.target.value });
  };

  // Handler untuk submit form
  const handleSubmit = async (e) => {
    e.preventDefault();
    setLoading(true);
    setError('');
    
    try {
      // Kirim permintaan ke API untuk menghasilkan pertanyaan
      const response = await axios.post('/api/questions/generate', formData, {
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${localStorage.getItem('token')}` // Token autentikasi
        }
      });
      
      // Simpan pertanyaan yang dihasilkan ke state
      setGeneratedQuestions(response.data.data);
    } catch (error) {
      // Tangani error
      setError(error.response?.data?.message || 'Failed to generate questions');
    } finally {
      setLoading(false);
    }
  };

  // Handler untuk menyetujui pertanyaan
  const handleApprove = async (questionId) => {
    try {
      // Kirim permintaan ke API untuk menyetujui pertanyaan
      await axios.put(`/api/questions/${questionId}/approve`, {}, {
        headers: {
          'Authorization': `Bearer ${localStorage.getItem('token')}` // Token autentikasi
        }
      });
      
      // Update state lokal
      setGeneratedQuestions(prevQuestions => 
        prevQuestions.map(q => 
          q._id === questionId ? { ...q, status: 'approved' } : q
        )
      );
    } catch (error) {
      // Tangani error
      setError(error.response?.data?.message || 'Failed to approve question');
    }
  };

  return (
    <div className="container mx-auto p-4">
      <h1 className="text-2xl font-bold mb-6">Generate Questions with AI</h1>
      
      {/* Form untuk menghasilkan pertanyaan */}
      <form onSubmit={handleSubmit} className="bg-white shadow-md rounded px-8 pt-6 pb-8 mb-4">
        <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
          {/* Input mata pelajaran */}
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
          
          {/* Dropdown kelas */}
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
          
          {/* Input lainnya (chapter, type, count, difficulty) */}
          
          {/* Tombol submit */}
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
      
      {/* Tampilkan error jika ada */}
      {error && <div className="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded mb-4">{error}</div>}
      
      {/* Tampilkan pertanyaan yang dihasilkan */}
      {generatedQuestions.length > 0 && (
        <div className="mt-8">
          <h2 className="text-xl font-bold mb-4">Generated Questions</h2>
          
          {/* Daftar pertanyaan */}
          {generatedQuestions.map((question, index) => (
            <div key={question._id} className="bg-white shadow-md rounded p-4 mb-4">
              {/* Header pertanyaan dengan tombol aksi */}
              <div className="flex justify-between items-start">
                <h3 className="font-bold">Question {index + 1}</h3>
                <div className="flex space-x-2">
                  {/* Tombol approve */}
                  <button
                    onClick={() => handleApprove(question._id)}
                    disabled={question.status === 'approved'}
                    className={`px-3 py-1 rounded text-white ${question.status === 'approved' ? 'bg-gray-400' : 'bg-green-500 hover:bg-green-600'}`}
                  >
                    {question.status === 'approved' ? 'Approved' : 'Approve'}
                  </button>
                  {/* Tombol reject dan edit */}
                </div>
              </div>
              
              {/* Konten pertanyaan */}
              <p className="my-2">{question.content}</p>
              
              {/* Tampilkan pilihan untuk pertanyaan pilihan ganda */}
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
              
              {/* Tampilkan untuk pertanyaan benar/salah dan essay */}
              
              {/* Tampilkan penjelasan */}
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

**Penjelasan Komponen:**
- Komponen ini menangani proses generasi pertanyaan dengan AI dari sisi frontend
- Menyediakan form untuk memasukkan parameter generasi pertanyaan (mata pelajaran, kelas, bab, tipe, jumlah, tingkat kesulitan)
- Mengirim permintaan ke API backend untuk menghasilkan pertanyaan
- Menampilkan pertanyaan yang dihasilkan dengan format yang sesuai berdasarkan tipe pertanyaan
- Menyediakan tombol untuk menyetujui, menolak, atau mengedit pertanyaan
- Menangani loading state dan error yang mungkin terjadi selama proses

Kode-kode di atas merupakan inti dari sistem ExamExpert AI dengan penjelasan detail untuk setiap fungsi, menunjukkan bagaimana sistem mengintegrasikan AI untuk menghasilkan pertanyaan, mengelola kuis, dan memungkinkan siswa untuk berpartisipasi dalam kuis.



        


          
Saya akan memeriksa file-file utama dalam proyek ExamExpert AI untuk memberikan penjelasan dalam bentuk kalimat tentang fungsi-fungsi utama sistem.
        
            
toolName: view_files
            
status: success
          
            
filePath: d:\Project\Tugas Akhir\backend-examexpert-ai\src\controllers\question.controller.js
          
Saya akan memeriksa file perplexity.js untuk memahami bagaimana sistem menggunakan AI untuk menghasilkan pertanyaan.
        
            
toolName: view_files
            
status: success
          
            
filePath: d:\Project\Tugas Akhir\backend-examexpert-ai\src\utils\perplexity.js
          
Saya akan memeriksa file quiz.controller.js untuk memahami bagaimana sistem mengelola kuis.
        
            
toolName: view_files
            
status: success
          
            
filePath: d:\Project\Tugas Akhir\backend-examexpert-ai\src\controllers\quiz.controller.js
          
Saya akan memeriksa file student.controller.js untuk memahami bagaimana sistem mengelola interaksi siswa dengan kuis.
        
            
toolName: view_files
            
status: success
          
            
filePath: d:\Project\Tugas Akhir\backend-examexpert-ai\src\controllers\student.controller.js
          
Saya akan memeriksa file question.model.js untuk memahami struktur data pertanyaan dalam sistem.
        
            
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
          
Saya akan memeriksa file auth.controller.js untuk memahami bagaimana sistem mengelola autentikasi pengguna.
        
            
toolName: view_files
            
status: success
          
            
filePath: d:\Project\Tugas Akhir\backend-examexpert-ai\src\controllers\auth.controller.js
          
Saya akan memeriksa file letterTemplates.js untuk memahami bagaimana sistem mengelola notifikasi email.
        
            
toolName: view_files
            
status: success
          
            
filePath: d:\Project\Tugas Akhir\backend-examexpert-ai\src\utils\letterTemplates.js
          
# Penjelasan Fungsi-Fungsi Utama ExamExpert AI

Berikut adalah penjelasan dalam bentuk kalimat untuk fungsi-fungsi utama dalam sistem ExamExpert AI:

## Fungsi Generasi Pertanyaan dengan AI (question.controller.js)

Fungsi `generateAIQuestions` dalam sistem ExamExpert AI bertugas menghasilkan pertanyaan menggunakan kecerdasan buatan. Fungsi ini menerima parameter seperti mata pelajaran, kelas, bab, jenis soal, jumlah soal, dan tingkat kesulitan dari pengguna. Setelah memvalidasi parameter yang diperlukan, fungsi ini memanggil utilitas Perplexity AI untuk menghasilkan pertanyaan sesuai dengan kriteria yang diminta. Pertanyaan yang dihasilkan kemudian disimpan dalam database dengan status "pending_review" yang menandakan bahwa pertanyaan tersebut perlu ditinjau sebelum digunakan dalam kuis. Fungsi ini hanya dapat diakses oleh pengguna dengan peran guru atau admin.

## Fungsi Integrasi dengan Perplexity API (perplexity.js)

Fungsi `generateQuestions` dalam file perplexity.js menangani komunikasi dengan API Perplexity untuk menghasilkan pertanyaan dalam bahasa Indonesia. Fungsi ini mengambil konfigurasi API dari variabel lingkungan, kemudian menyusun prompt yang sesuai berdasarkan jenis pertanyaan yang diminta (pilihan ganda, benar/salah, atau esai). Prompt ini berisi instruksi terperinci tentang format JSON yang diharapkan untuk setiap jenis pertanyaan. Setelah mengirim permintaan ke API Perplexity, fungsi ini memproses respons yang diterima dan mengembalikan array pertanyaan yang telah diformat dengan benar. Fungsi ini juga menangani kesalahan yang mungkin terjadi selama proses komunikasi dengan API.

## Fungsi Pembuatan Kuis (quiz.controller.js)

Fungsi `createQuiz` memungkinkan guru atau admin untuk membuat kuis baru dalam sistem. Fungsi ini memvalidasi parameter kuis seperti judul, mata pelajaran, kelas, bab, dan daftar pertanyaan. Setelah memastikan semua pertanyaan yang dipilih ada dalam database, fungsi ini membuat entri kuis baru dengan kode akses yang dihasilkan secara otomatis. Kuis yang dibuat dikaitkan dengan pengguna yang membuatnya, dan informasi kuis ditambahkan ke daftar kuis yang dibuat oleh pengguna tersebut. Fungsi ini juga mengirimkan email konfirmasi kepada guru yang membuat kuis, yang berisi detail kuis termasuk kode akses yang dapat dibagikan kepada siswa.

## Fungsi Bergabung dengan Kuis untuk Siswa (student.controller.js)

Fungsi `joinQuiz` memungkinkan siswa untuk bergabung dengan kuis menggunakan kode akses. Fungsi ini memvalidasi kode akses yang dimasukkan, memeriksa apakah kuis aktif, dan memastikan kuis belum berakhir. Jika siswa sudah bergabung sebelumnya, fungsi ini mengembalikan detail kuis tanpa menambahkan siswa sebagai peserta baru. Jika siswa belum bergabung, fungsi ini menambahkan siswa ke daftar peserta kuis dengan status "joined". Detail kuis yang dikembalikan kepada siswa tidak menyertakan jawaban yang benar untuk menjaga integritas kuis.

## Model Data Pertanyaan (question.model.js)

Model `Question` mendefinisikan struktur data untuk pertanyaan dalam sistem. Model ini mencakup bidang-bidang penting seperti konten pertanyaan, jenis (pilihan ganda, benar/salah, esai), mata pelajaran, kelas, bab, tingkat kesulitan, pilihan jawaban, jawaban yang benar, penjelasan, pembuat pertanyaan, dan status. Model ini juga menyimpan informasi tentang persetujuan atau penolakan pertanyaan, termasuk siapa yang menyetujui atau menolak, kapan, dan alasan penolakan jika ada. Terdapat juga penanda apakah pertanyaan dihasilkan oleh AI dan daftar tag untuk kategorisasi. Model ini menggunakan indeks untuk pencarian yang efisien berdasarkan mata pelajaran, kelas, bab, jenis, dan tingkat kesulitan.

## Model Data Kuis (quiz.model.js)

Model `Quiz` mendefinisikan struktur data untuk kuis dalam sistem. Model ini mencakup bidang-bidang seperti judul, deskripsi, mata pelajaran, kelas, bab, daftar pertanyaan, pembuat kuis, kode akses yang dihasilkan secara otomatis, durasi, status aktif, tanggal mulai, dan tanggal berakhir. Model ini juga menyimpan daftar peserta kuis, yang mencakup informasi tentang setiap siswa yang bergabung, termasuk waktu bergabung, waktu mulai, status, dan jawaban yang diberikan. Untuk setiap jawaban, model menyimpan pertanyaan yang dijawab, jawaban yang dipilih, dan apakah jawaban tersebut benar.

## Model Data Pengguna (user.model.js)

Model `User` mendefinisikan struktur data untuk pengguna dalam sistem. Model ini mencakup bidang-bidang seperti nama, email, kata sandi (yang di-hash), peran (admin, guru, siswa), dan status. Model ini juga menyimpan informasi administratif seperti siapa yang menyetujui atau menolak pengguna, kapan, dan alasan penolakan jika ada. Untuk pengguna dengan peran guru, model menyimpan informasi tambahan seperti institusi, keahlian, pengalaman, dan dokumen verifikasi. Model ini juga menangani enkripsi kata sandi menggunakan bcrypt.

## Fungsi Autentikasi (auth.controller.js)

Fungsi `register` menangani pendaftaran pengguna baru dalam sistem. Fungsi ini memvalidasi data pengguna seperti nama, email, kata sandi, dan peran, memeriksa apakah pengguna sudah ada, dan membuat pengguna baru dengan peran default "siswa" jika tidak ada peran yang ditentukan. Setelah membuat pengguna, fungsi ini mengirimkan email selamat datang dan mengembalikan data pengguna beserta token JWT untuk autentikasi. Fungsi `generateToken` digunakan untuk membuat token JWT berdasarkan ID pengguna, yang digunakan untuk autentikasi dalam sistem.

## Fungsi Notifikasi Email (letterTemplates.js)

Fungsi `sendWelcomeLetter` mengirimkan email selamat datang kepada pengguna baru saat mereka mendaftar. Fungsi ini menghasilkan konten HTML yang disesuaikan berdasarkan peran pengguna (siswa, guru, atau admin), dengan penekanan pada fitur-fitur yang relevan untuk peran tersebut. Email ini memberikan pengantar tentang sistem ExamExpert AI dan membantu pengguna baru memahami fitur-fitur yang tersedia untuk mereka berdasarkan peran mereka.
        
