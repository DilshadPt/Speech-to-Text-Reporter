//# Speech-to-Text-Reporter
//An application coded in Python language which listens human speech and convert it into computer readable text which can be used for text editing and command purposes
from sys import byteorder
from array import array
from struct import pack
from tkFileDialog import askopenfilename

import time
from threading import *
import wx
import pyaudio
import wave
import os,sys
import Tkinter,tkFileDialog



THRESHOLD = 500
CHUNK_SIZE = 1024
FORMAT = pyaudio.paInt16
RATE = 16000
FLAG = 0 
path = 'sas1.wav'
path1 = 'comm.wav'
data = 0
sample_width = 0
count = 1	

temp = 1	
result = ''
b = ''
   
# Button definitions
ID_START = wx.NewId()
ID_STOP = wx.NewId()
ID_PROCESS = wx.NewId()
ID_TEX = wx.NewId()
ID_COMM = wx.NewId()
   
# Define notification event for thread completion
EVT_RESULT_ID = wx.NewId()
text_ctrl_1 = wx.NewId()
def EVT_RESULT(win, func):
       """Define Result Event."""
       win.Connect(-1, -1, EVT_RESULT_ID, func)
   
class ResultEvent(wx.PyEvent):
       print """Simple event to carry arbitrary result data."""
       def __init__(self, data):
           """Init Result Event."""
           wx.PyEvent.__init__(self)
           self.SetEventType(EVT_RESULT_ID)
           self.data = data
   
# Thread class that executes processing
class WorkerThread(Thread):
      
       
       print """Worker Thread Class."""
       def __init__(self, notify_window):
           """Init Worker Thread Class."""
           Thread.__init__(self)
           self._notify_window = notify_window
           self._want_abort = 0
           # This starts the thread running on creation, but you could
           # also make the GUI thread responsible for calling this
           self.start()
   
       def run(self):
       	   global result
       	   global result1
	   p = pyaudio.PyAudio()
	   stream = p.open(format=FORMAT, channels=1, rate=RATE,
		input=True, output=True,
		frames_per_buffer=CHUNK_SIZE)

	   num_silent = 0
           snd_started = False
	   r = array('h')
           """Run Worker Thread."""
           # This is the code executing in the new thread. Simulation of
           # a long process (well, 10s here) as a simple loop - you will
           # need to structure your processing so that you periodically
           # peek at the abort variable
           while 1:
               snd_data = array('h', stream.read(CHUNK_SIZE))
	       if byteorder == 'big':
		    snd_data.byteswap()
	       r.extend(snd_data)

	       time.sleep(0.01)
	       if self._want_abort:
	       	   
		   print "end"
                   sample_width = p.get_sample_size(FORMAT)
		   stream.stop_stream()
		   stream.close()
		   p.terminate()

		   
		   
		  
		   r = pack('<' + ('h'*len(r)), *r)
		   wf = wave.open(path, 'wb')
	           wf.setnchannels(1)
	           wf.setsampwidth(sample_width)
	           wf.setframerate(RATE)
	           wf.writeframes(r)
	           wf.close()
	          
	             	              
			   
		   try:

			import pocketsphinx 
			import sphinxbase
			hmdir = "/usr/share/pocketsphinx/model/hmm/wsj1"
			lmd = "/usr/share/pocketsphinx/model/lm/wsj/wlist5o.3e-7.vp.tg.lm.DMP"
			dictd = "/usr/share/pocketsphinx/model/lm/wsj/wlist5o.dic"
			speechRec = pocketsphinx.Decoder(hmm = hmdir, lm = lmd, dict = dictd)
			wavfile = "sas1.wav"
			wavFile = file(wavfile,'rb')
			wavFile.seek(44)
			speechRec.decode_raw(wavFile)
			result = speechRec.get_hyp()
			print result
			
		   except:
		   	self.count = 0
			import pocketsphinx 
			hmdir = "/usr/share/pocketsphinx/model/hmm/wsj1"
			lmd = "/usr/share/pocketsphinx/model/lm/wsj/wlist5o.3e-7.vp.tg.lm.DMP"
			dictd = "/usr/share/pocketsphinx/model/lm/wsj/wlist5o.dic"
			speechRec = pocketsphinx.Decoder(hmm = hmdir, lm = lmd, dict = dictd)
			wavfile = "sas1.wav"
			wavFile = file(wavfile,'rb')
			wavFile.seek(44)
			speechRec.decode_raw(wavFile)
			result = speechRec.get_hyp()
			
			print result
				
	           
	           
		   return format(result)
		   # Use a result of None to acknowledge the abort (of
                   # course you can use whatever you'd like or even
                   # a separate event type)
                  
                  
           # Here's where the result would be returned (this is an
           # example fixed result of the number 10, but it could be
           # any Python object)
	   '''wx.PostEvent(self._notify_window, ResultEvent(None))
           wx.PostEvent(self._notify_window, ResultEvent(1))'''
   
       def abort(self):
           print """abort worker thread."""
           # Method for use by main thread to signal an abort
           self._want_abort = 1
           
   
   # GUI Frame class that spins off the worker thread
class MainFrame(wx.Frame):
       """Class MainFrame."""
       
       
       def __init__(self, parent, id):
           """Create the MainFrame."""
           wx.Frame.__init__(self,parent, id, 'Thread Test')
   	   
	  

	   

           # Dumb sample frame with two buttons
           wx.Button(self, ID_START, 'Start', pos=(0,0))
           wx.Button(self, ID_PROCESS, 'process', pos=(0,50))
           wx.Button(self, ID_STOP, 'Stop', pos=(0,100))
           wx.Button(self, ID_TEX, 'Tex', pos=(0,150))
           wx.Button(self, ID_COMM, 'Command', pos=(0,200))
           
           self.status = wx.StaticText(self, -1, '', pos=(0,150))
           self.text_ctrl_1 = wx.TextCtrl(self, -1, "",pos=(200,200))
           self.SetSize((600, 329))
           self.text_ctrl_1.SetMinSize((300,100))
           
           
   
           self.Bind(wx.EVT_BUTTON, self.OnStart, id=ID_START)
           self.Bind(wx.EVT_BUTTON, self.OnStop, id=ID_STOP)
           self.Bind(wx.EVT_BUTTON, self.OnProcess, id=ID_PROCESS)
           self.Bind(wx.EVT_BUTTON, self.OnTex, id=ID_TEX)
   	   self.Bind(wx.EVT_BUTTON, self.OnComm, id=ID_COMM)
           # Set up event handler for any worker thread results
           EVT_RESULT(self,self.OnResult)
   
           # And indicate we don't have a worker thread yet
           self.worker = None
   
       def OnStart(self, event):
           print """Start Computation."""
           # Trigger the worker thread unless it's already busy
           if not self.worker:
               self.status.SetLabel('Starting computation')
               self.worker = WorkerThread(self)
               
       def OnProcess(self, event):
       	   global b
       	   print result
       	   res = str(result)
       	   a,b,c,d,e = res.split("'")
       	   
       	   print b
       	   
       	   
           self.status.SetLabel('Start Processing')
           self.text_ctrl_1.SetValue(b)
               
   
       def OnStop(self, event):
           """Stop Computation."""
           # Flag the worker thread to stop if running
           if self.worker:
               self.status.SetLabel('Trying to abort computation')
               self.worker.abort()
               self.worker = 0
               print "111111111111111111111111111111111111111111"
   
       def OnResult(self, event):
           print """Show Result status."""
           if event.data is None:
               # Thread aborted (using our convention of None return)
               self.status.SetLabel('Computation aborted')
           else:
              # Process results here
              self.status.SetLabel('Computation Result: %s' % event.data)
              # In either event, the worker is done
              self.worker = None
              
       def OnTex(self,event):
       	   text_file = open("Output.txt", "w")

	   text_file.write(b)
	   
	   text_file.close()
 	   os.system("gedit Output.txt")
       
       def OnComm(self,event):
       	   
       	   
       	    '''
       	    root = Tkinter.Tk() 
       	    root.withdraw() 
       	    dirs=tkFileDialog.asksaveasfile(mode='w')
       	    dirs.write("dfdfd")
       	   
	   
	    print dirs '''
           
            

	    def record():
		    p = pyaudio.PyAudio()
		    stream = p.open(format=FORMAT, channels=1, rate=RATE,
			input=True, output=True,
			frames_per_buffer=CHUNK_SIZE)

		    num_silent = 0
		    

		    r = array('h')

	  	    while 1:
				# little endian, signed short
				snd_data = array('h', stream.read(CHUNK_SIZE))
				if byteorder == 'big':
				    snd_data.byteswap()
				r.extend(snd_data)

				

				
				num_silent += 1
				
				if num_silent > 50:
				    break

		    sample_width = p.get_sample_size(FORMAT)
		    stream.stop_stream()
		    stream.close()
		    p.terminate()

		    
		    return sample_width, r

	   
	    sample_width, data = record()
            data = pack('<' + ('h'*len(data)), *data)

	    wf = wave.open(path1, 'wb')
	    wf.setnchannels(1)
	    wf.setsampwidth(sample_width)
	    wf.setframerate(RATE)
	    wf.writeframes(data)
	    wf.close() 
	    
	    try:

			import pocketsphinx 
			import sphinxbase
			hmdir = "/usr/share/pocketsphinx/model/hmm/wsj1"
			lmd = "/usr/share/pocketsphinx/model/lm/wsj/wlist5o.3e-7.vp.tg.lm1.DMP"
			dictd = "/usr/share/pocketsphinx/model/lm/wsj/wlist5o1.dic"
			speechRec = pocketsphinx.Decoder(hmm = hmdir, lm = lmd, dict = dictd)
			wavfile = "comm.wav"
			wavFile = file(wavfile,'rb')
			wavFile.seek(44)
			speechRec.decode_raw(wavFile)
			result1 = speechRec.get_hyp()
			print result1
			
	    except:
		   	self.count = 0
			import pocketsphinx 
			hmdir = "/usr/share/pocketsphinx/model/hmm/wsj1"
			lmd = "/usr/share/pocketsphinx/model/lm/wsj/wlist5o.3e-7.vp.tg.lm1.DMP"
			dictd = "/usr/share/pocketsphinx/model/lm/wsj/wlist5o1.dic"
			speechRec = pocketsphinx.Decoder(hmm = hmdir, lm = lmd, dict = dictd)
			wavfile = "comm.wav"
			wavFile = file(wavfile,'rb')
			wavFile.seek(44)
			speechRec.decode_raw(wavFile)
			result1 = speechRec.get_hyp()
			
			print result1
	  
	    res1 = str(result1)
       	    h,i,j,k,m = res1.split("'")
       	   
       	    print i
	    if i == 'SAVE':
	    	    	
	       	    root = Tkinter.Tk() 
	       	    root.withdraw() 
	       	    dirs=tkFileDialog.asksaveasfile(mode='w')
	       	    dirs.write(b)
	       	    
	       	    print dirs
           	 
	    if i == 'OPEN':
	   	    root = Tkinter.Tk(); 
	   	    root.withdraw()
   		    filename = askopenfilename(parent=root)
		    filename

		    f=open(filename)
		    res = f.read()
		    print res
		    	
		    f.close()
		    self.text_ctrl_1.SetValue(res)
		    
		    
	    if i == 'CLEAR':
	    	    self.text_ctrl_1.Clear() 
       	   
class MainApp(wx.App):
             """Class Main App."""
             def OnInit(self):
		     """Init Main App."""
		     self.frame = MainFrame(None, -1)
		     self.frame.Show(True)
		     self.SetTopWindow(self.frame)
		     return True
  
if __name__ == '__main__':
         app = MainApp(0)
         app.MainLoop()
