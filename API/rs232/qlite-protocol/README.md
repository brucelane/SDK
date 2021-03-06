The nsISystemAdapterSerial Interface allows to manage serial port. QLite protocol is an example of using nsISystemAdapterSerial:

    <html xmlns="http://www.w3.org/1999/xhtml"
           xmlns:html="http://www.w3.org/1999/xhtml">
    <head>
      <style>
        * { background-color: white }
      </style>
      <title>NEC</title>
      <script type="text/javascript" language="JavaScript">
    //<![CDATA[
    const Ci=Components.interfaces;
    const PR_RDONLY = 0x1;
    const nsISystemAdapterSerial = Ci.nsISystemAdapterSerial;
    var gLogger; 

    const serialSysPath = "*";
    const id = 0;
    dump("log4Service = " + log4Service + "\n");
    gLogger = log4Service.getLogger("test.led");
    dump ("gLogger = " + gLogger + "\n");
    LOG("TEST");
    function ERROR(string) 
    {
      gLogger.error (string, null);
      appendConsole(string + "\n");
      //  dump("*** ERROR MIRE *** " + string + "\n");
    } 
    function LOG(string) 
    {
      gLogger.debug (string, null);
      appendConsole(string + "\n");
       // dump("*** LOG MIRE *** " + string + "\n");
    } 
    function MLNProtocol()
    {
       this.logger = log4Service.getLogger("mln.test");
    }

    MLNProtocol.prototype =
    {
     PROTO_A: 0xAA,
     PROTO_B: 0xBB,
     SID_POS:2,
        HEADER_LENGTH:9,
        LENGTH_POS:6,
        HEADER_CS_POS:8,
        ID_MULTICAST:0xFF,
     FC_POS:9,
        FC_CODES: {
      SendToInitialSegment:0x00,
      SetClock:0x01,
      SendToSegment:0x02,
      SendContinueTable:0x03,
      SendToInsertSegment:0x05,
      Ping:0x20,
      RequestDisplayTransmission:0x21,
      ResponseDisplayTransmission:0x31,
      RequestDisplayStatus:0x22,
      ResponseDisplayStatus:0x32,
     },
        CC_CODES: {
      ClearDisplay:0x01,
      ShowTextString:0x03,
      ShowCurrentTime:0x04,
      ShowCurrentDate:0x05,
      Delay:0x06,
      DisplayBuffer:0x07,
      ShowTextImmediately:0x08,
      EndOfSegmentData:0x1F,
     },
     ID_BROADCAST:255,
        STATUS: {
      Normal: 0x60,
      TimeoutError: 0x61,
      ChecksumError: 0x62,
      OverflowError: 0x63,
      FunctioncodeError: 0x64,
      ParameterError: 0x65,
     },
     MODE: {
      Instant:0x00,
      ScanRight:0x01,
      ScanLeft:0x02,
      ScanDown:0x03,
      ScanUp:0x04,
      ShiftRight:0x05,
      ShiftLeft:0x06,
     },
     SPEED: {
      Speed0:0,
      Speed1:1,
      Speed2:2,
      Speed3:3,
      Speed4:4,
      Speed5:5,
      Speed6:6,
      Speed7:7,
      Speed8:8,
      Speed9:9,
     },
     COLOR: {
      Black: 0x00,
      Red: 0x01,
      Green: 0x02,
      Yellow: 0x03,
     },
     logger:null,
        serial: null,
     inited: false,
        inputStream: null,
        outputStream: null,
     senderId:0,
        receiverId:0,
     msg:null,
        offset:0,
     fc:-1,
     length:0,
     tcp:false,
     padding:0,
        LOG: function(string) 
     {
      this.logger.debug (string, null);
      if (LOG != undefined && typeof LOG == "function")
       LOG(string);
     }, 
     ERROR: function(string) 
     {
      this.logger.error (string, null);
      if (ERROR != undefined && typeof ERROR == "function")
       ERROR(string);
     },
        initWithSerial: function(aSerial)
        {
            this.serial = aSerial;
      this.serial.recieveTimeout = 1000;
      this.serial.recieveMode = nsISystemAdapterSerial.RECIEVE_MODE_SYNC;
      this.serial.setConfig(nsISystemAdapterSerial.DIRECTION_IN|nsISystemAdapterSerial.DIRECTION_OUT,
              nsISystemAdapterSerial.BAUD9600,8,
              nsISystemAdapterSerial.PARITY_NONE,
             nsISystemAdapterSerial.STOPBIT_1,
             nsISystemAdapterSerial.FLOWCONTROL_OFF);  
      this.serial.open();
      this.outputStream = this.serial.outputStream;
      this.inputStream = this.serial.inputStream;
      this.inited = true;
      this.tcp = false;
      this.LOG("initWithSerial OK\n");
        },
     initForTcp: function(host, port)
     {
      netscape.security.PrivilegeManager.enablePrivilege("UniversalXPConnect");
    const BinaryInputStream = CC("@mozilla.org/binaryinputstream;1",
                                  "nsIBinaryInputStream",
                                  "setInputStream");
      var options=["starttls"];
      var transport = Cc["@mozilla.org/network/socket-transport-service;1"]
                .getService(Ci.nsISocketTransportService)
          .createTransport(null, 0, host, port, null);
      //var socketInputStream = transport.openInputStream(Ci.nsITransport.OPEN_BLOCKING, 0, 0);
      var socketInputStream = transport.openInputStream(0, 0, 0);
      var socketOutputStream = transport.openOutputStream( Ci.nsITransport.OPEN_BLOCKING, 0, 0);
      this.outputStream =  socketOutputStream;
      this.inputStream = new BinaryInputStream(socketInputStream);
      this.inited = true;
      this.tcp = true;
      this.LOG("initForTcp OK\n");
     },
     deinit: function()
     {
      this.outputStream =  null;
      this.inputStream =  null;
      if (this.serial != null)
      {
       this.serial.close();
                this.serial = null;
      }
      this.inited = false;
     },
     setSenderId : function (aId)
     {
      this.senderId=aId;
     },
     setReceiverId : function (aId)
     {
      this.receiverId=aId;
     },
     addPadding: function (aPadding)
     {
      this.padding = aPadding;
     },
     send: function()
     {
      if (this.tcp)
       netscape.security.PrivilegeManager.enablePrivilege("UniversalXPConnect");
            if (!this.inited)
      {
       throw "Protocol not inited";
      }
            if (this.msg == null)
      {
       throw "Message not inited";
      }
      if (this.fc == -1)
      {
       throw "no function code";
      }
      if (this.padding != 0)
      {
       for (i = 0; i <this.padding; i++)
       {
        this.msg[this.length++] = 0x41;
       }
      }
      var array = this.msg.subarray(0, this.length);
      this.LOG ("send '" + this.toHexaString(this.msg, 0, this.length) + "'");
      var str = String.fromCharCode.apply(null,array);
      this.outputStream.write(str, this.length);
            var drain = true;
      if (drain)
      {
       this.serial.drainTQ();
      }
      this.LOG("send OK\n");
     },
     receive: function()
     {
            if (!this.inited)
      {
       throw "Protocol not inited";
      }
      try {
       this.LOG("receive before\n");
       var b = this.inputStream.readByteArray(this.HEADER_LENGTH + 1); 
       this.LOG("receive after b.length = " + b.length);
       if (b.length != this.HEADER_LENGTH + 1)
        return null;
       this.LOG("header = " + this.toHexaString(b, 0, b.length));
                var length = this.getShortBE(b, this.LENGTH_POS);
       var fc = b[this.FC_POS];
       this.LOG("receive OK length=" + length + " fc = " + this.toHexa(fc));
       b = this.inputStream.readByteArray(length);
       if (b.length != length)
        throw "can't read message body";
                this.LOG("body = " + this.toHexaString(b, 0, b.length));
       var obj = {};
       obj.fc = fc;
       obj.data = b;
       if  (fc == this.FC_CODES.ResponseDisplayTransmission)
       {
        this.Get_ResponseDisplayTransmission(obj);
       }
       else if  (fc == this.FC_CODES.ResponseDisplayStatus)
       {
        this.Get_ResponseDisplayStatus(obj);
       }
       return obj;
      }
      catch (ex) 
      {
       LOG("Exception " + ex);                                                         return null;                                
      }
     },
     beginMessage: function()
        {
            if (!this.inited)
      {
       throw "Protocol not inited";
      }
      this.msg = new Uint8Array(1024*64);
      this.padding = 0;
      this.offset = 0;
      this.fc = -1;
      this.msg[this.offset++] = this.PROTO_A;
      this.msg[this.offset++] = this.PROTO_B;
      this.setShortBE(this.msg, this.offset, this.senderId); this.offset += 2;
      this.setShortBE(this.msg, this.offset, this.receiverId); this.offset += 2;
        },

        endMessage: function()
     {
            if (!this.inited)
      {
       throw "Protocol not inited";
      }
      if (this.msg == null)
      {
       throw "Message not inited";
      }
      if (this.fc == -1)
      {
       throw "no function code";
      }
      var length = this.offset;
            this.LOG("endMessage this.offset = " + this.offset);
      this.setShortBE(this.msg, this.LENGTH_POS, length - this.FC_POS);
      this.setCheckSum(this.msg, this.SID_POS, this.HEADER_CS_POS);
      this.setCheckSum(this.msg, this.FC_POS, this.offset++);
      this.length = this.offset;
     },
        FC_SendToInitialSegment: function ()
     {
      this.fc = this.FC_CODES.SendToInitialSegment;
      this.offset=this.FC_POS;
      this.msg[this.offset++] = this.fc;
     },
        FC_SetClock: function (date)
     {
      this.fc = this.FC_CODES.SetClock;
      this.offset=this.FC_POS;
      this.msg[this.offset++] = this.fc;
      this.msg[this.offset++] = date.getFullYear() - 2000;
      this.msg[this.offset++] = date.getMonth() + 1;
      this.msg[this.offset++] = date.getUTCDate();
      this.msg[this.offset++] = date.getDay(); 
      this.msg[this.offset++] = date.getHours();
      this.msg[this.offset++] = date.getMinutes();
      this.msg[this.offset++] = date.getSeconds();
            this.LOG("FC_SetClock this.offset = " + this.offset);
     },
     FC_SendToSegment: function (numSegment)
     {
      this.fc = this.FC_CODES.SendToSegment;
      this.offset=this.FC_POS;
      this.msg[this.offset++] = this.fc;
      this.msg[this.offset++] = numSegment;
     },
     FC_SendToInsertSegment: function (delayMinute)
     {
      this.fc = this.FC_CODES.SendToInsertSegment;
      this.offset=this.FC_POS;
      this.msg[this.offset++] = this.fc;
      this.msg[this.offset++] = delayMinute;
     },
     FC_SendContinueTable: function (firstSegment, lastSegment)
     {
      this.fc = this.FC_CODES.SendContinueTable;
      this.offset=this.FC_POS;
      this.msg[this.offset++] = this.fc;
      this.msg[this.offset++] = firstSegment;
      this.msg[this.offset++] = lastSegment;
     },
     FC_RequestDisplayStatus: function()
     {
      this.fc = this.FC_CODES.RequestDisplayStatus;
      this.offset=this.FC_POS;
      this.msg[this.offset++] = this.fc;
      
     },
     FC_Ping: function()
     {
      this.fc = this.FC_CODES.Ping;
      this.offset=this.FC_POS;
      this.msg[this.offset++] = this.fc;
     },
     FC_RequestDisplayTransmission: function()
     {
      this.fc = this.FC_CODES.RequestDisplayTransmission;
      this.offset=this.FC_POS;
      this.msg[this.offset++] = this.fc;
      
     },
     Get_ResponseDisplayTransmission: function(object)
     {
      object.status = object.data[0];
      this.LOG("Get_ResponseDisplayTransmission status = " + this.toHexa(object.status));
     },
     Get_ResponseDisplayStatus: function(object)
     {
      var offset = 0;
      object.Columns = this.getShortLE(object.data, offset); offset += 2;
      object.Scanheight = object.data[offset++];
      object.Rows = object.data[offset++];
      object.FPGA_Scanheight = object.data[offset++];
      object.FPGA_Rows = object.data[offset++];
      
      this.LOG("Get_ResponseDisplayStatus \nColumns = " + object.Columns +
        "\nScanheight = " + object.Scanheight +
        "\nRows = " + object.Rows +
        "\nFPGA_Scanheight = " + object.FPGA_Scanheight +
         "\nFPGA_Rows = " + object.FPGA_Rows);
     },
     CC_ClearDisplay: function ()
     {
      if (this.fc != this.FC_CODES.SendToInitialSegment &&
         this.fc != this.FC_CODES.SendToSegment &&
         this.fc != this.FC_CODES.SendToInsertSegment)
      {
       throw "CC ClearDisplay not valid from fc = " + this.fc;
            }
      this.cc = this.CC_CODES.ClearDisplay;
      this.LOG("CC_ClearDisplay this.offset = " + this.offset + " cc = " + this.cc);
      this.msg[this.offset++] = this.cc;
     },
     /** @param aDelay delay to wait in seconds */
     CC_Delay: function (aDelay)
     {
      if (this.fc != this.FC_CODES.SendToInitialSegment &&
         this.fc != this.FC_CODES.SendToSegment &&
         this.fc != this.FC_CODES.SendToInsertSegment)
      {
       throw "CC_Delay not valid from fc = " + this.fc;
            }
      this.cc = this.CC_CODES.Delay;
      this.LOG("CC_Delay this.offset = " + this.offset + " cc = " + this.cc);
      this.msg[this.offset++] = this.cc;
      this.setShortLE(this.msg, this.offset, aDelay); this.offset+= 2;
     },
     CC_EndOfSegmentData: function ()
     {
      if (this.fc != this.FC_CODES.SendToSegment &&
         this.fc != this.FC_CODES.SendToInitialSegment &&
         this.fc != this.FC_CODES.SendToInsertSegment)
      {
       throw "CC_EndOfSegmentDatanot valid from fc = " + this.fc;
            }
      this.cc = this.CC_CODES.EndOfSegmentData;
      this.LOG("CC_EndOfSegmentData this.offset = " + this.offset + " cc = " + this.cc);
      this.msg[this.offset++] = this.cc;
     },
     CC_ShowTextString: function(aFont, aFcolor, aBcolor, aX, aY, aText)
     {
      if (this.fc != this.FC_CODES.SendToInitialSegment &&
         this.fc != this.FC_CODES.SendToSegment &&
         this.fc != this.FC_CODES.SendToInsertSegment)
      {
       throw "CC_ShowTextString not valid from fc = " + this.fc;
            }
      this.cc = this.CC_CODES.ShowTextString;
      this.msg[this.offset++] = this.cc;
      this.setByte(this.msg, this.offset++, aFont);
      this.setByte(this.msg, this.offset++, aFcolor);
      this.setByte(this.msg, this.offset++, aBcolor);
      this.setByte(this.msg, this.offset++, aX);
      this.setByte(this.msg, this.offset++, aY);
      this.setStr(this.msg, this.offset, aText); this.offset += aText.length;
      this.msg[this.offset++] = 0x00;
     },
     CC_ShowCurrentTime: function(aFormat, aFont, aFcolor, aBcolor, aX, aY)
     {
      if (this.fc != this.FC_CODES.SendToInitialSegment &&
         this.fc != this.FC_CODES.SendToSegment &&
         this.fc != this.FC_CODES.SendToInsertSegment)
      {
       throw "CC_ShowCurrentTime not valid from fc = " + this.fc;
            }
      this.cc = this.CC_CODES.ShowCurrentTime;
      this.msg[this.offset++] = this.cc;
      this.setByte(this.msg, this.offset++, aFormat);
      this.setByte(this.msg, this.offset++, aFont);
      this.setByte(this.msg, this.offset++, aFcolor);
      this.setByte(this.msg, this.offset++, aBcolor);
      this.setByte(this.msg, this.offset++, aX);
      this.setByte(this.msg, this.offset++, aY);
     },
     CC_ShowCurrentDate: function(aFormat, aFont, aFcolor, aBcolor, aX, aY)
     {
      if (this.fc != this.FC_CODES.SendToInitialSegment &&
         this.fc != this.FC_CODES.SendToSegment &&
         this.fc != this.FC_CODES.SendToInsertSegment)
      {
       throw "CC_ShowCurrentDate not valid from fc = " + this.fc;
            }
      this.cc = this.CC_CODES.ShowCurrentDate;
      this.msg[this.offset++] = this.cc;
      this.setByte(this.msg, this.offset++, aFormat);
      this.setByte(this.msg, this.offset++, aFont);
      this.setByte(this.msg, this.offset++, aFcolor);
      this.setByte(this.msg, this.offset++, aBcolor);
      this.setByte(this.msg, this.offset++, aX);
      this.setByte(this.msg, this.offset++, aY);
     },
     CC_DisplayBuffer: function(aDelay, aMode, aSpeed)
     {
      if (this.fc != this.FC_CODES.SendToInitialSegment &&
         this.fc != this.FC_CODES.SendToSegment &&
         this.fc != this.FC_CODES.SendToInsertSegment)
      {
       throw "CC_DisplayBuffer not valid from fc = " + this.fc;
            }
      this.testSpeed(aSpeed);
      this.testMode(aMode);
      this.cc = this.CC_CODES.DisplayBuffer;
      this.msg[this.offset++] = this.cc;
      this.setShortLE(this.msg, this.offset, aDelay); this.offset+= 2;
      this.setByte(this.msg, this.offset++, aMode);
      this.setByte(this.msg, this.offset++, aSpeed);
     },
     CC_ShowTextImmediately: function(aMode, aSpeed, aFont, aFcolor, aBcolor, aX, aY, aText)
     {
      this.LOG("CC_ShowTextImmediately");
      if (this.fc != this.FC_CODES.SendToInitialSegment &&
         this.fc != this.FC_CODES.SendToSegment &&
         this.fc != this.FC_CODES.SendToInsertSegment)
      {
       throw "CC_ShowTextImmediately not valid from fc = " + this.fc;
            }
      this.LOG("CC_ShowTextImmediately this.offset = " + this.offset);
      this.testMode(aMode);
      this.testSpeed(aSpeed);
      this.cc = this.CC_CODES.ShowTextImmediately;
      this.msg[this.offset++] = this.cc;
      this.setByte(this.msg, this.offset++, aMode);
      this.setByte(this.msg, this.offset++, aSpeed);
      this.setByte(this.msg, this.offset++, aFont);
      this.setByte(this.msg, this.offset++, aFcolor);
      this.setByte(this.msg, this.offset++, aBcolor);
      this.setByte(this.msg, this.offset++, aX);
      this.setByte(this.msg, this.offset++, aY);
      this.LOG("CC_ShowTextImmediately aText = '" + aText + "'");
      this.setStr(this.msg, this.offset, aText); this.offset += aText.length;
      this.msg[this.offset++] = 0x00;
     },
     setStr: function (data, offset, str)
     {
      var l= str.length;
      for (i = 0; i < l; i++)
      {
       data[offset+i]  = str.charCodeAt(i);
      }
     },
     testSpeed : function (speed)
     {
      if (speed < 0 || speed > 9)
      {
       throw "bad speed";
      }
     },
     testMode : function (mode)
     {
      if (mode < this.MODE.Instant || mode > this.MODE.ShiftLeft)
      {
       throw "bad mode";
      }
     },
     setCheckSum: function (data, start, offset)
     {
      data[offset]=0;
      for (i = start; i < offset; i++)
      {
       data[offset]  = data[offset] ^ data[i];
      }
      this.LOG("setCheckSum : " + this.toHexa(data[offset]));
     },
     setByte: function (data, offset, val)
     {
      var v = val & 0xFF;
      data[offset] = val;
     },
     getByte: function (data,offset)
     {
      return data;
     },
     setShortBE: function (data, offset, val)
     {
      data[offset] = (val>>8) & 0xFF;
      data[offset+1] = val & 0xFF;
     },
     getShortBE: function (data,offset)
     {
      return data[offset] << 8 | data[offset+1];
     },
     setShortLE: function (data, offset, val)
     {
      data[offset+1] = (val>>8) & 0xFF;
      data[offset] = val & 0xFF;
     },
     getShortLE: function (data,offset)
     {
      return data[offset+1] << 8 | data[offset];
     },
     toHexaString: function (data, start, length)
     {
      var str="";
      for (i = start; i < length; i++)
      {
       str += this.toHexa(data[i]);
       str +="|";
      }
      return str;
     },
     toHexa: function(val)
     {
      var str="";
      if (val == undefined)
       str += 'UU';
      else 
      {
       var sh = val.toString(16);
       if (sh.length == 1)
        sh = "0" + sh;
       str = sh.toUpperCase();
      }
      return str;
     }
    }
    function MLNHelper (serialSysPath, receiverId)
    {
     this.serialSysPath = serialSysPath;
     this.receiverId = receiverId;
     var mln = new MLNProtocol();
     this.COLOR = mln.COLOR;
     this.MODE = mln.MODE;
    }
    MLNHelper.prototype =
    {
     sendScrollText: function (text, color, speed, waitAfter)
     {
      LOG("sendScrollText");
      var serial = this._getSerial(this.serialSysPath);
      serial.close();
      var mln = new MLNProtocol();
      mln.setReceiverId(this.receiverId);
      mln.initWithSerial(serial);
      this._sendPing(mln);
      this._sendScrollText(mln, text, color, speed, 0);
      this._sendDisplayTransmission(mln);
      mln.deinit();
     },
     sendText: function (text, mode, color, speed)
     {
      LOG("sendText");
      var serial = this._getSerial(this.serialSysPath);
      serial.close();
      var mln = new MLNProtocol();
      mln.setReceiverId(this.receiverId);
      mln.initWithSerial(serial);
      this._sendPing(mln);
      this._sendText(mln, text, mode, color, speed);
      this._sendDisplayTransmission(mln);
      //this._sendContinueTable(mln);
      mln.deinit();
     },
     sendText2Page: function (text1, text2, mode, color, speed)
     {
      LOG("sendText");
      var serial = this._getSerial(this.serialSysPath);
      serial.close();
      var mln = new MLNProtocol();
      mln.setReceiverId(this.receiverId);
      mln.initWithSerial(serial);
      this._sendPing(mln);
      mln.beginMessage();
      mln.FC_SendToSegment(0);
      mln.CC_ClearDisplay();
      mln.CC_ShowTextString(0, color,0,0,0, text1);
      //mln.CC_DisplayBuffer(0, mode, speed);
    // Affichage du premier text pendant 2 secondes
      mln.CC_DisplayBuffer(2000, mode, speed);
    // Effacement du test
      mln.CC_ClearDisplay();
      mln.CC_ShowTextString(0, color,0,0,0, "            ");
      mln.CC_DisplayBuffer(0, mln.MODE.Instant, speed);
      mln.CC_ShowTextString(0, color,0,0,0, text2);
      //mln.CC_DisplayBuffer(0, mode, speed);
      mln.CC_DisplayBuffer(2000, mode, speed);
      mln.CC_ShowTextString(0, color,0,0,0, "            ");
      mln.CC_DisplayBuffer(0, mln.MODE.Instant, speed);
      mln.CC_EndOfSegmentData();
      mln.endMessage();
      mln.addPadding(64);
      mln.send();
      //this._sendDisplayTransmission(mln);
      mln.deinit();
     },
     sendDate: function (date)
     {
      LOG("sendTime");
      var serial = this._getSerial(this.serialSysPath);
      serial.close();
      var mln = new MLNProtocol();
      mln.setReceiverId(this.receiverId);
      mln.initWithSerial(serial);
    // Change date
      mln.beginMessage();
      mln.FC_SetClock(date);
      mln.endMessage();
      mln.addPadding(64);
      mln.send();
      mln.deinit();
     },
     sendTime: function (color)
     {
      LOG("sendTime");
      var serial = this._getSerial(this.serialSysPath);
      serial.close();
      var mln = new MLNProtocol();
      mln.setReceiverId(this.receiverId);
      mln.initWithSerial(serial);
      this._sendPing(mln);
    // Show clock during 10 s
      mln.beginMessage();
      mln.FC_SendToSegment(0);
      mln.CC_ClearDisplay();
      mln.CC_ShowCurrentTime(1, 0, color,0,0,0);
      mln.CC_DisplayBuffer(10000, mln.MODE.Instant, 9);
    // Show date during 10 s
      mln.CC_ClearDisplay();
      mln.CC_ShowCurrentDate(2, 0, color,0,0,0);
      mln.CC_DisplayBuffer(10000, mln.MODE.Instant, 9);
      mln.CC_EndOfSegmentData();
      mln.endMessage();
      mln.addPadding(64);
      mln.send();
      //this._sendDisplayTransmission(mln);
      mln.deinit();
     },
     sendClearDisplay: function()
     {
      var serial = this._getSerial(this.serialSysPath);
      serial.close();
      var mln = new MLNProtocol();
      mln.setReceiverId(this.receiverId);
      mln.initWithSerial(serial);
      this._sendPing(mln);
      this._sendClear(mln)
      //this._sendDisplayTransmission(mln);
      mln.deinit();
     },
     _getSerial: function (sysPath)
     {
      var system= systemManager;
      var serials = system.getAdaptersByIId(Ci.nsISystemAdapterSerial);
      LOG("serials.length = " + serials.length);
      for (i = 0; i < serials.length; i++)
      {
       serial = serials.queryElementAt(i,Ci.nsISystemAdapterSerial);
       LOG("serial.sysPath = " + serial.sysPath);
                if (sysPath == "*")
                    return serial;
       if (serial.sysPath == sysPath)
        return serial;
      }
      return null;
     },
     _sendPing: function(mln)
     {
      LOG("_sendPing");
      mln.beginMessage();
      mln.FC_Ping();
      mln.endMessage();
      mln.send();
      var resp = mln.receive();
      if (resp == null || resp.fc != mln.FC_CODES.Ping)
      {
       throw "bad response for Ping";
      }
      else
      {
       LOG ("Ping OK");
      }
     },
     _sendStatus: function (mln)
     {
      LOG("_sendStatus");
      mln.beginMessage();
      mln.FC_RequestDisplayStatus();
      mln.endMessage();
      mln.send();
      var resp = mln.receive();
      if (resp == null || resp.fc != mln.FC_CODES.ResponseDisplayStatus)
      {
       throw "bad response for RequestDisplayStatus";
      }
      else
      {
       LOG ("_sendStatus OK");
       return resp;
      }
     },
     _sendContinueTable: function (mln)
     {
      LOG("_sendContinueTable");
      mln.beginMessage();
      mln.FC_SendContinueTable();
      mln.endMessage();
      mln.send();
      this._sendDisplayTransmission(mln);
     },
     _sendDisplayTransmission: function (mln)
     {
      LOG("_sendDisplayTransmission");
      mln.beginMessage();
      mln.FC_RequestDisplayTransmission();
      mln.endMessage();
      mln.send();
      var resp = mln.receive();
      if (resp == null || resp.fc != mln.FC_CODES.ResponseDisplayTransmission)
      {
       throw "bad response for RequestDisplayTransmission";
      }
      else if (resp.status != mln.STATUS.Normal)
      {
       throw "bad status in ResponseDisplayTransmission status = " + mln.toHexa(resp.status);
      }
      else
      {
       LOG ("RequestDisplayTransmission OK");
      }
     },
     _sendText: function (mln, text, mode, color, speed)
     {
      LOG("_sendText");
      mln.beginMessage();
      mln.FC_SendToSegment(0);
      mln.CC_ClearDisplay();
    // Ligne 1
      mln.CC_ShowTextString(0, color,0,0,0, text);
    // Ligne 2
      mln.CC_ShowTextString(0, color,0,0,1, text);
      //mln.CC_DisplayBuffer(0, mln.MODE.ScanRight, 9);
      //mln.CC_DisplayBuffer(0, mln.MODE.Instant, 9);
      //mln.CC_DisplayBuffer(0, mln.MODE.Instant, 9);
      mln.CC_DisplayBuffer(2000, mode , speed);
      mln.CC_EndOfSegmentData();
      mln.endMessage();
      mln.addPadding(64);
      mln.send();
     },
     _sendScrollText: function (mln, text, color, speed, waitAfter)
     {
      LOG("_sendScrollText");
      var status = this._sendStatus(mln);
      var lineWidth = status.Columns/6;
      mln.beginMessage();
      var initial = false;
      var insert = false;
      if (initial)
    // Initial segment will be remplaced by the insert and normal segment
       mln.FC_SendToInitialSegment();
      else if (insert)
    // Insert segment during 1 minutes
       mln.FC_SendToInsertSegment(1);
      else
    // Normal segment (persistent)
       mln.FC_SendToSegment(0);
      var l = text.length;
      l = (lineWidth>l) ? lineWidth : l;
      var blanks="";
    // Delete prior text
      for (i = 0; i < l; i++) blanks += " ";
      mln.CC_ShowTextImmediately(mln.MODE.Instant, 9, 0, color,0,0,0, blanks);
    // Complete with blank
      l = lineWidth;
      for (i = 0; i < l; i++)
       text += " ";
    // Show new text
      mln.CC_ShowTextImmediately(mln.MODE.ShiftLeft, speed, 0, color,0,0,0, text);
      if (waitAfter != 0)
      {
       mln.CC_Delay(waitAfter*1000);
      }
      mln.CC_EndOfSegmentData();
      mln.endMessage();
      mln.addPadding(64);
      mln.send();
     },
     _sendClear: function (mln)
     {
      LOG("_sendClear");
      mln.beginMessage();
      mln.FC_SendToSegment();
      mln.CC_ClearDisplay();
      mln.CC_EndOfSegmentData();
      mln.endMessage();
      mln.send();
     },
    }  
    function changeDisplay ()
    {
     var text = document.getElementById("textChange");
     LOG("changeDisplay");
     try {
      var helper = new MLNHelper(serialSysPath, id);
      helper.sendScrollText(text.value, helper.COLOR.Red, 8, 0);
     }
     catch (ex)
     {
      ERROR ("Exception  " + ex + " line = "+ ex.lineNumber);
     }
    }
    function clearDisplay()
    {
     LOG("clearDisplay");
     try {
      var helper = new MLNHelper(serialSysPath, id);
      helper.sendClearDisplay();
     }
     catch (ex)
     {
      ERROR ("Exception  " + ex + " line = "+ ex.lineNumber);
     }
    }
    function sleep(milliseconds)                            
    {                                                                               
     netscape.security.PrivilegeManager.enablePrivilege("UniversalXPConnect");
      // We basically just call this once after the specified number of milliseconds
     LOG("sleep " + milliseconds + " milliseconds");
     var timeup = false;                                                    
     function wait() { timeup = true; }     
     window.setTimeout(wait, milliseconds);
        
     var thread = Components.classes["@mozilla.org/thread-manager;1"].
            getService().currentThread;                         
     while(!timeup) {                        
      thread.processNextEvent(true);
     }                               
     LOG("sleep end");
    }
    function init()
    {
     LOG("init");
     try {
      var helper = new MLNHelper(serialSysPath, id);
      var text = "test titi ligne tres longue NNNNNNNNNNNNNNNNNNNNNNNNNNN";
      document.getElementById("textChange").value = text;
      var test = "texte scroll";
      if (test == "date")
      {
       var date = new Date();
       date.setMonth(date.getMonth() - 0);
       helper.sendDate(date);
       helper.sendTime(helper.COLOR.Red);
      }
      else if (test == "texte")
      {
       helper.sendText("test2", helper.MODE.ScanRight, helper.COLOR.Yellow, 9);
      }
      else if (test == "texte scroll")
      {
       helper.sendScrollText(text, helper.COLOR.Red, 3, 0);
      }
      else if (test == "texte 2")
      {
       helper.sendText2Page("page1", "Page2", helper.MODE.ScanLeft, helper.COLOR.Red, 3);
      }
      else 
       helper.sendClearDisplay();
     }
     catch (ex)
     {
      ERROR ("Exception  " + ex + " line = "+ ex.lineNumber);
     }
    }
    function deinit()
    {
     LOG("deinit");
     try
     {
     }
     catch(e){
      LOG ("Exception  " + e + "line = "+ e.lineNumber);
     }
    }

    function do_check_true(cond, text) {
      // we don't have the test harness' utilities in this scope, so we need this
      // little helper. In the failure case, the exception is propagated to the
      // caller in the main run_test() function, and the test fails.
      if (!cond)
        throw "Failed check: " + text;
    }
    function bin2String(array) {
      return String.fromCharCode.apply(String,array);
    }
    function readBytes(inputStream, count)
    {
      LOG("readBytes before");
      var bi = new BinaryInputStream(inputStream);
      LOG("readBytes inputStream = " + inputStream + "bi=" + bi);
      var buf = new BinaryInputStream(inputStream).readByteArray(count);
      LOG("readBytes buf=" + buf);
      return buf;
    }
    function appendConsole(str)
    {
         var console=document.getElementById("console")
         if (console)
             console.value =console.value + str;
    }
    function clearConsole(str)
    {
         var console=document.getElementById("console")
         if (console)
             console.value ="";
    }

    //]]>
     </script>
    <body onload="setTimeout('init()', '10')" onunload="deinit();" _onfocus="init()">
     <table border="0">
       <tr>
            <td><textarea id="console" cols="80" rows="20"> </textarea></td>
       </tr>
       <tr>
            <td><input id="textChange" type="text" size="80"/></td>
            <td><input id="change" type="button" value="Change display" onclick="changeDisplay()"></td>
          </tr>
       <tr>
            <td><input id="clear" type="button" value="Clear display"  onclick="clearDisplay()"></td>
          </tr>
       <tr>
            <td><input id="clearConsole" type="button" value="Clear console"  onclick="clearConsole()"></td>
          </tr>
       </table>
    </body>
    </html>
