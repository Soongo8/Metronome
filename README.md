import SwiftUI
import AVFoundation

// MARK: - [1] 메트로놈의 뇌 (로직 + 오디오 엔진 통합)
class MetronomeViewModel: ObservableObject {
    // --- [기존 변수들] ---
    @Published var bpm: Int = 120
    @Published var isPlaying: Bool = false
    @Published var isSoundOn: Bool = true
    @Published var isSilentPreBeat: Bool = false
    @Published var rhythm: String = "4분음표"
    @Published var timeSignature: Int = 4
    @Published var muteProbability: Double = 0.0
    @Published var currentBeatIndex: Int = 0
    
    // --- [내부 로직용 변수] ---
    private var timer: Timer?
    private var subdivisionCounter: Int = 0
    private var lastTapTime: Date?
    private var tapIntervals: [TimeInterval] = []
    
    // --- [오디오 엔진 변수] ---
    private var engine: AVAudioEngine!
    private var playerNode: AVAudioPlayerNode!
    private var audioFile: AVAudioFile?
    private var audioBuffer: AVAudioPCMBuffer?
    
    init() {
        setupAudio()
    }
    
    // MARK: - 오디오 엔진 설정 (포맷 매칭 적용)
    private func setupAudio() {
        engine = AVAudioEngine()
        playerNode = AVAudioPlayerNode()
        
        // 1. 오디오 파일 먼저 로드
        guard let url = Bundle.main.url(forResource: "beat", withExtension: "mp3") else {
            print("❌ 오디오 파일(beat.mp3)을 찾을 수 없습니다.")
            return
        }
        
        do {
            // 2. 파일 로드 및 포맷 확인
            audioFile = try AVAudioFile(forReading: url)
            let format = audioFile!.processingFormat
            
            print("📊 오디오 포맷: \(format.channelCount) 채널, \(format.sampleRate) Hz")
            
            // 3. 노드 연결 시 파일의 포맷 사용 (핵심!)
            engine.attach(playerNode)
            engine.connect(playerNode, to: engine.mainMixerNode, format: format)
            
            // 4. 버퍼로 로드
            let frameCount = UInt32(audioFile!.length)
            audioBuffer = AVAudioPCMBuffer(pcmFormat: format, frameCapacity: frameCount)
            if let buffer = audioBuffer {
                try audioFile!.read(into: buffer)
                print("✅ 오디오 버퍼 로드 완료")
            }
            
            // 5. 엔진 준비 및 시작
            engine.prepare()
            try engine.start()
            print("✅ 오디오 엔진 준비 완료")
        } catch {
            print("❌ 오디오 설정 실패: \(error)")
        }
    }
    
    // MARK: - 재생용 함수 (정확한 타이밍 스케줄링)
    private func playSoundEffect() {
        guard let buffer = audioBuffer else {
            print("⚠️ 오디오 버퍼가 없습니다.")
            return
        }
        
        // 엔진이 꺼져있다면 켭니다.
        if !engine.isRunning {
            do {
                try engine.start()
                // 엔진이 막 시작되었으므로 약간의 딜레이 후 재생
                DispatchQueue.main.asyncAfter(deadline: .now() + 0.01) {
                    self.playSoundEffect()
                }
                return
            } catch {
                print("❌ 오디오 엔진 시작 실패: \(error)")
                return
            }
        }
        
        // 플레이어 노드가 실행 중이 아니면 시작
        if !playerNode.isPlaying {
            playerNode.play()
        }
        
        // 정확한 타이밍을 위한 미래 시간 계산
        guard let lastRenderTime = playerNode.lastRenderTime else {
            // 엔진이 준비되지 않았으면 즉시 재생
            playerNode.scheduleBuffer(buffer, at: nil, options: .interrupts, completionHandler: nil)
            return
        }
        
        // 최소 딜레이로 스케줄링 (약 10ms)
        let delaySeconds: Double = 0.01
        let sampleRate = lastRenderTime.sampleRate
        let delayFrames = AVAudioFramePosition(delaySeconds * sampleRate)
        let startTime = AVAudioTime(sampleTime: lastRenderTime.sampleTime + delayFrames, atRate: sampleRate)
        
        // 미래 시간에 스케줄링
        playerNode.scheduleBuffer(buffer, at: startTime, options: .interrupts, completionHandler: nil)
    }

    // MARK: - 메트로놈 제어 (Start/Stop)
    
    func start() {
        isPlaying = true
        timer?.invalidate()
        
        currentBeatIndex = 0
        subdivisionCounter = 0
        isSilentPreBeat = false
        
        // 엔진과 플레이어 노드 준비
        if !engine.isRunning {
            do {
                try engine.start()
            } catch {
                print("❌ 오디오 엔진 시작 실패: \(error)")
                return
            }
        }
        
        // 플레이어 노드가 중지된 상태라면 다시 시작 준비
        if !playerNode.isPlaying {
            playerNode.play()
        }
        
        DispatchQueue.main.asyncAfter(deadline: .now() + 0.05) {
            // 1. 첫 박자 즉시 실행
            self.tick()
            
            // 2. 타이머 시작 (반복)
            let baseInterval = 60.0 / Double(self.bpm)
            let divider = self.getDivider(for: self.rhythm)
            let actualInterval = baseInterval / divider
            
            self.timer = Timer.scheduledTimer(withTimeInterval: actualInterval, repeats: true) { [weak self] _ in
                self?.tick()
            }
        }
    }

    func stop() {
        isPlaying = false
        isSilentPreBeat = false
        timer?.invalidate()
        timer = nil
        currentBeatIndex = 0
        subdivisionCounter = 0
        // playerNode.stop()을 제거 - 재생 시 문제 발생 방지
        // 대신 엔진은 계속 실행 상태로 유지
    }
    
    func togglePlay() {
        if isPlaying { stop() } else { start() }
    }
    
    func toggleSound() {
        isSoundOn.toggle()
    }
    
    // MARK: - 박자 실행 로직 (Tick)
    private func tick() {
        // [1] 침묵 예비박 처리
        if isSilentPreBeat {
            isSilentPreBeat = false
            currentBeatIndex = 0
            subdivisionCounter = 0
            return
        }
        
        // [2] 본 연주 소리 재생
        let randomValue = Double.random(in: 0...100)
        let isRandomMuted = randomValue < muteProbability
        
        if isSoundOn && !isRandomMuted {
            playSoundEffect()
        }
        
        // [3] 비주얼 불빛 로직
        let divider = Int(getDivider(for: rhythm))
        
        subdivisionCounter += 1
        
        if subdivisionCounter >= divider {
            subdivisionCounter = 0
            DispatchQueue.main.async {
                self.currentBeatIndex = (self.currentBeatIndex + 1) % self.timeSignature
            }
        }
    }
    
    // --- [기존 헬퍼 함수들 유지] ---
    private func getDivider(for rhythm: String) -> Double {
        switch rhythm {
        case "4분음표": return 1.0
        case "8분음표": return 2.0
        case "3연음": return 3.0
        case "16분음표": return 4.0
        default: return 1.0
        }
    }

    func tapTempo() {
        let now = Date()
        if let lastTime = lastTapTime {
            let interval = now.timeIntervalSince(lastTime)
            if interval > 2.0 {
                tapIntervals.removeAll()
            } else {
                tapIntervals.append(interval)
                if tapIntervals.count > 4 { tapIntervals.removeFirst() }
                let averageInterval = tapIntervals.reduce(0, +) / Double(tapIntervals.count)
                let newBPM = Int(60.0 / averageInterval)
                self.bpm = max(1, min(300, newBPM))
                if isPlaying { start() }
            }
        }
        lastTapTime = now
    }
    
    func adjustBPM(by amount: Int) {
        bpm = max(1, min(300, bpm + amount))
        if isPlaying { start() }
    }
    
    func restartIfPlaying() {
        if isPlaying { start() }
    }
    
    deinit {
        stop()
        engine.stop()
    }
}

// MARK: - [2] 앱의 얼굴 (UI - Grooove 스타일 적용)
struct ContentView: View {
    @StateObject private var vm = MetronomeViewModel()
    
    // UI에 사용할 커스텀 색상 정의
    let darkRed = Color(red: 61/255, green: 20/255, blue: 20/255)
    let brownColor = Color(red: 95/255, green: 58/255, blue: 58/255)
    let blueColor = Color(red: 0/255, green: 122/255, blue: 255/255)

    var body: some View {
        ZStack {
            // 1. 배경색 (흰색)
            Color.white.edgesIgnoringSafeArea(.all)
            
            VStack(spacing: 30) {
                // 2. 타이틀
                Text("Grooove")
                    .font(.system(size: 40, weight: .bold, design: .rounded))
                    .foregroundColor(.black)
                    .padding(.top, 50)
                
                // 3. 박자표 선택
                Picker("Time Signature", selection: $vm.timeSignature) {
                    Text("2/4").tag(2)
                    Text("3/4").tag(3)
                    Text("4/4").tag(4)
                }
                .pickerStyle(SegmentedPickerStyle())
                .background(Color.gray.opacity(0.2))
                .cornerRadius(8)
                .padding(.horizontal, 40)
                .onChange(of: vm.timeSignature) { _ in vm.restartIfPlaying() }
                
                // 4. 비주얼 인디케이터
                HStack(spacing: 12) {
                    ForEach(0..<vm.timeSignature, id: \.self) { index in
                        Circle()
                            .frame(width: 25, height: 25)
                            .foregroundColor(
                                vm.currentBeatIndex == index && !vm.isSilentPreBeat
                                ? .orange
                                : .gray.opacity(0.3)
                            )
                            .animation(.easeInOut(duration: 0.1), value: vm.currentBeatIndex)
                            .shadow(
                                color: (vm.currentBeatIndex == index && !vm.isSilentPreBeat) ? .orange : .clear,
                                radius: 8
                            )
                    }
                }
                .padding(.vertical, 8)
                
                // 5. 리듬 선택 (커스텀 세그먼트 컨트롤 스타일)
                HStack(spacing: 0) {
                    RhythmButton(title: "Quarter", tag: "4분음표", currentRhythm: $vm.rhythm, color: brownColor, vm: vm)
                    RhythmButton(title: "Eighth", tag: "8분음표", currentRhythm: $vm.rhythm, color: brownColor, vm: vm)
                    RhythmButton(title: "Triplets", tag: "3연음", currentRhythm: $vm.rhythm, color: brownColor, vm: vm)
                    RhythmButton(title: "Sixteenth", tag: "16분음표", currentRhythm: $vm.rhythm, color: brownColor, vm: vm)
                }
                .background(brownColor)
                .cornerRadius(8)
                .padding(.horizontal, 20)
                
                // 6. BPM 조절부
                HStack(spacing: 30) {
                    // 마이너스 버튼
                    Button(action: { vm.adjustBPM(by: -1) }) {
                        Image(systemName: "minus.circle")
                            .resizable()
                            .frame(width: 40, height: 40)
                            .foregroundColor(.black)
                    }
                    
                    // BPM 표시 창
                    Text("\(vm.bpm)")
                        .font(.system(size: 40, weight: .bold))
                        .foregroundColor(.white)
                        .frame(width: 100, height: 60)
                        .background(brownColor)
                        .cornerRadius(12)
                    
                    // 플러스 버튼
                    Button(action: { vm.adjustBPM(by: 1) }) {
                        Image(systemName: "plus.circle")
                            .resizable()
                            .frame(width: 40, height: 40)
                            .foregroundColor(.black)
                    }
                }
                
                // 7. 랜덤 뮤트 슬라이더
                VStack(spacing: 10) {
                    Text("Random Mute: \(Int(vm.muteProbability))%")
                        .foregroundColor(.black)
                        .font(.headline)
                    
                    Slider(value: $vm.muteProbability, in: 0...100, step: 5)
                        .accentColor(blueColor)
                }
                .padding(.horizontal, 30)
                
                Spacer()
                
                // 8. 하단 버튼 (Tap & Play)
                HStack(spacing: 30) {
                    // Tap 버튼
                    Button(action: {
                        vm.tapTempo()
                    }) {
                        Text("Tap")
                            .font(.title2)
                            .fontWeight(.bold)
                            .foregroundColor(.white)
                            .frame(width: 120, height: 80)
                            .background(blueColor)
                            .cornerRadius(15)
                    }
                    
                    // Play/Stop 버튼
                    Button(action: {
                        vm.togglePlay()
                    }) {
                        Text(vm.isPlaying ? "Stop" : "Play")
                            .font(.title2)
                            .fontWeight(.bold)
                            .foregroundColor(.white)
                            .frame(width: 120, height: 80)
                            .background(vm.isPlaying ? Color.red : blueColor)
                            .cornerRadius(15)
                    }
                }
                .padding(.bottom, 50)
            }
        }
    }
}

// 리듬 선택을 위한 커스텀 버튼 컴포넌트
struct RhythmButton: View {
    let title: String
    let tag: String
    @Binding var currentRhythm: String
    let color: Color
    let vm: MetronomeViewModel
    
    var body: some View {
        Button(action: {
            currentRhythm = tag
            vm.restartIfPlaying()
        }) {
            Text(title)
                .font(.system(size: 14, weight: .medium))
                .padding(.vertical, 12)
                .frame(maxWidth: .infinity)
                .foregroundColor(currentRhythm == tag ? .black : .white)
                .background(currentRhythm == tag ? Color.white : color)
        }
    }
}

#Preview {
    ContentView()
}
