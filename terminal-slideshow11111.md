import os
import time
import sys
import termios
import tty
from typing import List, Dict

class TerminalSlideshow:
    def __init__(self):
        self.slides: List[Dict[str, str]] = []
        self.current_slide = 0
        
    def clear_screen(self):
        os.system('cls' if os.name == 'nt' else 'clear')
        
    def add_slide(self, title: str, content: str):
        self.slides.append({
            'title': title,
            'content': content
        })
        
    def draw_box(self, text: str, width: int) -> str:
        lines = text.split('\n')
        # Unicode box drawing characters
        top_border = '╭' + '─' * (width - 2) + '╮'
        bottom_border = '╰' + '─' * (width - 2) + '╯'
        
        result = [top_border]
        for line in lines:
            # Pad the line to width - 2 (accounting for borders)
            padded_line = line.ljust(width - 2)
            result.append('│' + padded_line + '│')
        result.append(bottom_border)
        
        return '\n'.join(result)

    def get_key(self):
        fd = sys.stdin.fileno()
        old_settings = termios.tcgetattr(fd)
        try:
            tty.setraw(sys.stdin.fileno())
            ch = sys.stdin.read(1)
            if ch == '\x1b':
                ch2 = sys.stdin.read(1)
                ch3 = sys.stdin.read(1)
                return {'[A': 'up', '[B': 'down',
                       '[C': 'right', '[D': 'left'}.get(ch2+ch3, None)
            return ch
        finally:
            termios.tcsetattr(fd, termios.TCSADRAIN, old_settings)
        
    def format_slide(self, slide: Dict[str, str]) -> str:
        terminal_width = os.get_terminal_size().columns
        content_width = min(terminal_width - 4, 80)  # Max width of 80 chars
        
        lines = []
        
        # Add title in a box
        title_box = self.draw_box(slide['title'], content_width)
        lines.extend(title_box.split('\n'))
        lines.append("")
        
        # Add content in a box
        content_box = self.draw_box(slide['content'], content_width)
        lines.extend(content_box.split('\n'))
        
        # Add navigation help
        lines.append("")
        nav_help = "← → นำทาง | q เพื่อออก"
        lines.append(nav_help.center(terminal_width))
        
        return "\n".join(lines)
    
    def show_current_slide(self):
        self.clear_screen()
        if 0 <= self.current_slide < len(self.slides):
            print(self.format_slide(self.slides[self.current_slide]))
            
    def run(self):
        while True:
            self.show_current_slide()
            key = self.get_key()
            
            if key in ['right', 'down']:
                self.current_slide = min(self.current_slide + 1, len(self.slides) - 1)
            elif key in ['left', 'up']:
                self.current_slide = max(self.current_slide - 1, 0)
            elif key == 'q':
                self.clear_screen()
                break

# Create presentation
presentation = TerminalSlideshow()

# Add slides with Thai content
presentation.add_slide(
    "การเปรียบเทียบในบริบทสังคม",
    """1. "ร้านอาหารนี้__(แออัด)ในวันหยุดสุดสัปดาห์มากกว่าวันธรรมดา"
2. "นั่นเป็นงานปาร์ตี้ที่__(สนุก)ที่สุดที่ฉันเคยไปในปีนี้"
3. "คาเฟ่ใหม่ชงกาแฟได้__(ดี)กว่าร้านเก่า"

คำตอบ: แออัดกว่า, สนุกที่สุด, ดีกว่า"""
)

presentation.add_slide(
    "การเปรียบเทียบเชิงคุณภาพ",
    """เพิ่มคำขยายการเปรียบเทียบที่เหมาะสม:
    
1. "แล็ปท็อปใหม่__ แพงกว่าที่คาดไว้"
2. "ฉบับร่างที่สอง__ ดีกว่าฉบับแรก"
3. "อากาศวันนี้__ เย็นกว่าเมื่อวาน"
4. "หลักสูตรขั้นสูง__ ท้าทายกว่าระดับเริ่มต้น"
5. "การจราจร__ แย่กว่าปกติ"

คำขยายที่เป็นไปได้: มาก, อย่างมาก, เล็กน้อย, พอสมควร, ค่อนข้าง"""
)

presentation.add_slide(
    "โครงสร้างการเปรียบเทียบซับซ้อน",
    """ยิ่ง... ยิ่ง...:
1. "ยิ่ง_ ฝึกฝน ยิ่ง_ พัฒนา"
2. "ยิ่ง_ เรียนรู้เกี่ยวกับโครงการ ยิ่ง_ ซับซ้อน"

สอง/สามเท่า... กว่า:
1. "รถคันนี้ราคา_ แพง_ รุ่นเก่า"
2. "ตึกใหม่_ สูง_ ตึกเก่า"

ไม่... เท่า:
1. "ภาคต่อไม่_ สนุก_ ภาคแรก"
2. "ผลลัพธ์ไม่_ น่าประทับใจ_ ที่เราหวัง" """
)

# Run the presentation
presentation.run()
