
LATEST VERSION IS 15 THIS IS BASICALLY THE STANDALONE VERSION WITH A FEW TWEAKS TO THE CODE
TO LOCK BUTTONS NOT IN USE AND CLEAR THE TEXT BOX IF YOU ARE FLASHING MULTIPLE UNITS.
*****************************************************************************************





IVE BEEN ADVISED TO ADD THE CODE FOR THIS PROJECT, I DONT KNOW WHERE IT GOES IN GITHUB
SO ITS AT THE BOTTOM OF THE README LATEST CODE BELOW IS FOR V15 FEEL FREE TO COMPILE IT 
YOURSELF 



*****************************************************************************************
FOR THOSE INTERESTED IN HOW THIS TOOL WORKS, IT SIMPLY HAS BUTTONS THAT YOU CLICK EACH BUTTON
RUNS EITHER A WEBLINK (TO LAUNCH A WEBPAGE) OR A BATCH FILE, YOU CAN SEE THE CONTENTS OF THE
BATCH FILES THEY ARE IN YOUR USER DIRECTORY IN P2PVFLASH FOLDER, JUST RIGHT CLICK AND CHOOSE 
EDIT OPTION, THERE IS NOTHING MALICIOUS BUT AS IT RUNS BATCH FILES SOME AV PROGRAMS WILL FLAG THIS
AS A POTENTIAL THREAT, IF YOU DONT WANT TO RUN THE EXE FILE THEN YOU CAN SIMPLY CLICK THE BATCH FILE
IN THE DIRECTORY MENTIONED ABOVE
******************************************************************************************

*****************************************************************************************

LATEST VERSION IS THE STANDALONE PRODUCT, THIS IS AS IT SOUNDS A SINGLE PROGRAM WITH ALL 
THE NEEDED TOOLS BUILT INTO IT SO NO INSTALLER ROUTINE JUST DOWNLOAD AND RUN, HOWEVER WINDOWS
AND SOME OTHER VIRUS SCANNERS DETECT IT AS A VIRUS DUE TO THE WAY THE SCRIPTS ARE RUN, IF YOU 
DONT LIKE THIS THEN USE V11 INSTEAD.
There are 2 versions one runs in powershell the other in good old DOS but both do the same thing
*******************************************************************************************



v11 is for windows users

IF YOU ARE A WIN7 USER PLEASE USE THAT VERSION AS THE V11 VERSION WONT WORK WITH WIN7

This is the Phantom 2 Vision Plus Flashing tool
this is designed only for the above quad and no others

Click the green code button then click download zip


Tool has a step by step guide please follow it.

You will need NetFramework installed to run it. this will install the
first time the tool is run

VIDEO GUIDE IS HERE https://www.youtube.com/watch?v=WGHUZOMq2w0&t=511s

********************************************************************************************
Imports System.ComponentModel
Imports System.IO
Imports System.IO.Ports
Imports System.Management
Imports System.Text.RegularExpressions
Imports System.Threading

Public Class Form1

    Private serialPort As New SerialPort()
    Private bootmeCount As Integer = 0
    Private maxBootmeCount As Integer = 4
    Private _comPorts As BindingList(Of ComPortInfo) = New BindingList(Of ComPortInfo)
    Private comPortPreviouslyDetected As Boolean = False ' Track COM port state

    Private Sub Form1_Load(sender As Object, e As EventArgs) Handles MyBase.Load
        UpdateCOM()
    End Sub

    Private Function KillProcessByName(processName As String) As Boolean
        Try
            For Each proc As Process In Process.GetProcessesByName(processName)
                proc.Kill()
                proc.WaitForExit()
            Next
            Return True
        Catch ex As Exception
            MessageBox.Show("Error killing process: " & ex.Message)
            Return False
        End Try
    End Function

    Private Sub GetComPorts()
        Dim portDict As New Dictionary(Of String, String)
        _comPorts.Clear()

        For Each pName As String In SerialPort.GetPortNames()
            If Not portDict.ContainsKey(pName) Then
                portDict.Add(pName, pName)
            End If
        Next

        Using searcher As New ManagementObjectSearcher("SELECT Name FROM Win32_PnPEntity WHERE PNPClass = 'Ports'")
            For Each obj As ManagementBaseObject In searcher.Get()
                Dim name As String = obj("Name")?.ToString()
                If Not String.IsNullOrEmpty(name) AndAlso name.ToUpper().Contains("COM") Then
                    Dim portName As String = name.Substring(name.IndexOf("(") + 1, name.IndexOf(")") - name.IndexOf("(") - 1)
                    If Not portDict.ContainsKey(portName) Then
                        portDict.Add(portName, name)
                    Else
                        portDict(portName) = name
                    End If
                End If
            Next
        End Using

        Dim selectedPort As String = Nothing
        For Each kvp As KeyValuePair(Of String, String) In portDict
            If kvp.Value.ToUpper().Contains("USB") Then
                selectedPort = kvp.Key
                Exit For
            End If
        Next

        If selectedPort Is Nothing AndAlso portDict.Count > 0 Then
            selectedPort = portDict.Keys.First()
        End If

        If selectedPort IsNot Nothing Then
            Environment.SetEnvironmentVariable("port", selectedPort, EnvironmentVariableTarget.Process)
            Environment.SetEnvironmentVariable("port", selectedPort, EnvironmentVariableTarget.User)

            If TextBoxDetectedPort.InvokeRequired Then
                TextBoxDetectedPort.Invoke(Sub() TextBoxDetectedPort.Text = "Cool! I have detected a USB on port " & selectedPort & ". Please continue.")
            Else
                TextBoxDetectedPort.Text = "Detected USB Port: " & selectedPort
            End If

            LogMsg("Auto-selected USB Port: " & selectedPort)

            Button3.Invoke(Sub() Button3.Enabled = True)
            Button4.Invoke(Sub() Button4.Enabled = True) ' enable button but leave text untouched

            comPortPreviouslyDetected = True
        Else
            ' No COM port detected
            If TextBoxDetectedPort.InvokeRequired Then
                TextBoxDetectedPort.Invoke(Sub() TextBoxDetectedPort.Text = "       Please plug USB TTL adapter into Computer")
            Else
                TextBoxDetectedPort.Text = "No USB Port detected"
            End If
            LogMsg("No USB Port found.")

            Button3.Invoke(Sub() Button3.Enabled = False)
            Button4.Invoke(Sub() Button4.Enabled = False)

            ' Clear RichTextBox when USB is unplugged
            RichTextBoxOutput.Invoke(Sub() RichTextBoxOutput.Clear())

            comPortPreviouslyDetected = False
        End If
    End Sub

    Public Sub UpdateCOM()
        Dim threadProc As New Thread(Sub()
                                         GetComPorts()
                                     End Sub)
        threadProc.SetApartmentState(ApartmentState.STA)
        threadProc.Start()
    End Sub

    Protected Overrides Sub WndProc(ByRef m As Message)
        If m.Msg = UsbDeviceNotification.WM_DEVICECHANGE Then
            Select Case CInt(m.WParam)
                Case UsbDeviceNotification.DbtDeviceRemoveComplete, UsbDeviceNotification.DbtDeviceArrival
                    UpdateCOM()
            End Select
        End If
        MyBase.WndProc(m)
    End Sub

    Private Sub LogMsg(msg As String, Optional includeTimestamp As Boolean = True)
        If includeTimestamp Then
            msg = $"{DateTime.Now:yyyy/MM/dd HH:mm:ss.fff} - {msg}"
        End If
        Debug.WriteLine(msg)
    End Sub

    Private Sub Button1_Click(sender As Object, e As EventArgs) Handles Button1.Click
        Try
            Dim tempPath As String = Path.Combine(Path.GetTempPath(), "driver.exe")
            File.WriteAllBytes(tempPath, My.Resources.usb)

            Dim startInfo As New ProcessStartInfo(tempPath) With {
                .UseShellExecute = True,
                .Verb = "runas",
                .WorkingDirectory = Path.GetTempPath()
            }

            Process.Start(startInfo)

        Catch ex As Win32Exception
            If ex.NativeErrorCode = 1223 Then
                MessageBox.Show("Driver installation was cancelled by the user.", "Cancelled", MessageBoxButtons.OK, MessageBoxIcon.Information)
            Else
                MessageBox.Show("An error occurred while trying to run the driver installer:" & vbCrLf & ex.Message, "Error", MessageBoxButtons.OK, MessageBoxIcon.Error)
            End If
        Catch ex As Exception
            MessageBox.Show("Unexpected error: " & ex.Message, "Error", MessageBoxButtons.OK, MessageBoxIcon.Error)
        End Try
    End Sub

    Private Sub Button2_Click(sender As Object, e As EventArgs) Handles Button2.Click
        Dim img As Image = My.Resources.wiring
        Dim tempPath As String = Path.Combine(Path.GetTempPath(), "wiring.jpg")
        img.Save(tempPath, System.Drawing.Imaging.ImageFormat.Jpeg)
        Process.Start(tempPath)
    End Sub

    Private Sub Button3_Click(sender As Object, e As EventArgs) Handles Button3.Click
        Dim detectedPort As String = Environment.GetEnvironmentVariable("port")

        If String.IsNullOrEmpty(detectedPort) Then
            MessageBox.Show("No COM port detected.")
            Return
        End If

        Try
            If serialPort.IsOpen Then serialPort.Close()

            With serialPort
                .PortName = detectedPort
                .BaudRate = 115200
                .DataBits = 8
                .StopBits = StopBits.One
                .Parity = Parity.None
                .Handshake = Handshake.None
                .ReadTimeout = 1000
                .WriteTimeout = 1000
            End With

            AddHandler serialPort.DataReceived, AddressOf SerialDataReceived
            serialPort.Open()

            RichTextBoxOutput.Clear()
            RichTextBoxOutput.AppendText("Listening on " & detectedPort & "..." & Environment.NewLine)

            bootmeCount = 0

        Catch ex As Exception
            MessageBox.Show("Error opening serial port: " & ex.Message)
        End Try
    End Sub

    Private Sub SerialDataReceived(sender As Object, e As SerialDataReceivedEventArgs)
        Try
            Dim data As String = serialPort.ReadExisting()
            If Not String.IsNullOrEmpty(data) Then
                If RichTextBoxOutput.InvokeRequired Then
                    RichTextBoxOutput.Invoke(Sub()
                                                 RichTextBoxOutput.AppendText(data)
                                                 RichTextBoxOutput.ScrollToCaret()
                                             End Sub)
                Else
                    RichTextBoxOutput.AppendText(data)
                    RichTextBoxOutput.ScrollToCaret()
                End If

                Dim occurrences As Integer = Regex.Matches(data.ToUpper(), "BOOTME").Count
                If occurrences > 0 Then
                    bootmeCount += occurrences
                    If bootmeCount >= maxBootmeCount Then
                        serialPort.Close()
                        If RichTextBoxOutput.InvokeRequired Then
                            RichTextBoxOutput.Invoke(Sub()
                                                         RichTextBoxOutput.AppendText(Environment.NewLine & "If BOOTME shows above you are good to flash if it doesnt check your wiring or if using pogo check your connections the board should have a green light on it." & Environment.NewLine)
                                                     End Sub)
                        Else
                            RichTextBoxOutput.AppendText(Environment.NewLine & "If BOOTME shows above you are good to flash." & Environment.NewLine)
                        End If
                    End If
                End If
            End If
        Catch ex As Exception
            ' Ignore
        End Try
    End Sub

    Public Class ComPortInfo
        Public Property Caption As String
        Public Property PortName As String
    End Class

    ' --- Button4 Flash Wi-Fi Module ---
    Private Sub Button4_Click(sender As Object, e As EventArgs) Handles Button4.Click
        Dim port As String = Environment.GetEnvironmentVariable("port")

        If String.IsNullOrEmpty(port) Then
            MessageBox.Show("No COM port set in environment.")
            Return
        End If

        Dim originalText As String = Button4.Text
        Button4.Enabled = False
        Button4.Text = "Now Flashing Board on " & port & "..."
        Application.DoEvents()

        Dim flashThread As New Thread(Sub()
                                          Try
                                              RichTextBoxOutput.Invoke(Sub()
                                                                           RichTextBoxOutput.Clear()
                                                                           RichTextBoxOutput.AppendText("Flashing in process please wait" & Environment.NewLine)
                                                                       End Sub)

                                              Dim userProfile As String = Environment.GetFolderPath(Environment.SpecialFolder.UserProfile)
                                              Dim workingDir As String = Path.Combine(userProfile, "p2vpflash")
                                              Directory.CreateDirectory(workingDir)

                                              File.WriteAllBytes(Path.Combine(workingDir, "ubl1.img"), My.Resources.ubl1)
                                              File.WriteAllBytes(Path.Combine(workingDir, "uboot.img"), My.Resources.uboot)
                                              File.WriteAllBytes(Path.Combine(workingDir, "sfh.exe"), My.Resources.sfh)

                                              ' --- FIRST RUN (kill after 6 seconds) ---
                                              Dim args1 As String = $"/c cd /d ""{workingDir}"" && sfh.exe -nandflash -v -p {port} ubl1.img uboot.img"
                                              Dim cmd1 As New ProcessStartInfo("cmd.exe") With {
                                                  .Arguments = args1,
                                                  .UseShellExecute = True,
                                                  .WindowStyle = ProcessWindowStyle.Normal
                                              }

                                              Dim p1 As Process = Process.Start(cmd1)
                                              Thread.Sleep(6000)
                                              KillProcessByName("sfh")
                                              Thread.Sleep(500)

                                              ' --- SECOND RUN (wait until completion) ---
                                              Dim args2 As String = args1
                                              Dim cmd2 As New ProcessStartInfo("cmd.exe") With {
                                                  .Arguments = args2,
                                                  .UseShellExecute = True,
                                                  .WindowStyle = ProcessWindowStyle.Normal
                                              }

                                              Dim p2 As Process = Process.Start(cmd2)
                                              If p2 IsNot Nothing Then
                                                  p2.WaitForExit()
                                              End If

                                              ' Flash completed message
                                              RichTextBoxOutput.Invoke(Sub()
                                                                           RichTextBoxOutput.AppendText("Flashing of WIFI Module completed successfully" & Environment.NewLine)
                                                                       End Sub)

                                              ' Remove port environment variable
                                              Dim key = Microsoft.Win32.Registry.CurrentUser.OpenSubKey("Environment", writable:=True)
                                              If key IsNot Nothing Then
                                                  key.DeleteValue("port", False)
                                                  key.Close()
                                              End If

                                              ' Show completion on button for 2 seconds
                                              Button4.Invoke(Sub()
                                                                 Button4.Text = "Flashing completed ✅"
                                                             End Sub)
                                              Thread.Sleep(2000)

                                          Catch ex As Exception
                                              MessageBox.Show("Error: " & ex.Message, "Error", MessageBoxButtons.OK, MessageBoxIcon.Error)
                                          Finally
                                              Button4.Invoke(Sub()
                                                                 Button4.Text = originalText
                                                                 Button4.Enabled = False ' disable until COM re-detected
                                                             End Sub)
                                          End Try
                                      End Sub)

        flashThread.IsBackground = True
        flashThread.Start()
    End Sub

    Private Sub Button5_Click(sender As Object, e As EventArgs) Handles Button5.Click
        Process.Start("https://www.youtube.com/watch?v=WGHUZOMq2w0&t=511s")
    End Sub

    Private Sub Button6_Click(sender As Object, e As EventArgs) Handles Button6.Click
        Process.Start("https://sites.google.com/view/phantom-2-wifi-module-fix/home")
    End Sub

    Private Sub Button7_Click(sender As Object, e As EventArgs) Handles Button7.Click
        Process.Start("https://buymeacoffee.com/flymymavic")
    End Sub

End Class

*********************************************************************************************************************

