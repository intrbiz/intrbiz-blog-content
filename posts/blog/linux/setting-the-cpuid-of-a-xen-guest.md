---
Date: 2012-08-02
Author: Chris Ellis
Category: blog/linux
---
# Setting the CPUID of a XEN guest

After some reading around I've discovered it is possible to configure the CPU 
vendor and model information that a XEN guest sees.

Most Linux sysadmins will be familiar with `cat /proc/cpuinfo` to get 
information about the systems processors.  This gives information about the CPU 
vendor, model and features.  For example my desktop gives:

    [cellis@cedesktop ~]$ cat /proc/cpuinfo 
    processor       : 0
    vendor_id       : AuthenticAMD
    cpu family      : 15
    model           : 67
    model name      : AMD Athlon(tm) 64 X2 Dual Core Processor 5200+
    - snip -

This information is actually exposed via 
[cpuid instruction](http://en.wikipedia.org/wiki/CPUID).  This instruction takes 
a command in the `EAX` register and populates the `EAX`, `EBX`, `ECX` and `EDX` registers 
with the requested data.

If `EAX` is set to zero, the CPU vendor string is returned.  This is a 12 byte 
ASCII string.  Which is stored in the `EBX`, `EDX`, `ECX` (that is certainly logical, 
I suspect a hangup from the circuit complexity).

The CPU model string is a little more complex, it is a 48 byte ASCII string.  
This string is obtained by executing the `cpuid` instruction 3 times, with `EAX` 
set to: `0x80000002`, `0x80000003` and `0x80000004`.

XEN has the [cpuid config option](http://zhigang.org/wiki/XenCPUID), which 
defines the values of `EAX`, `EBX`, `ECX` and `EDX` for specific values of `EAX`.  
Lets take the following XEN config:

    cpuid=['0:eax=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx,ebx=01110010011101000110111001001001, 
    ecx=00000000000000000000000000000000,edx=00000000011110100110100101100010']

This causes the XEN guest to see the CPU vendor string 'Intrbiz', by providing 
the cpuid register values when EAX = 0.

As such, my XEN VM now reports:

    xenvm1:~ # cat /proc/cpuinfo
    processor       : 0
    vendor_id       : Intrbiz
    cpu family      : 0
    model           : 0
    model name      : Intrbiz XEN virtual CPU
    stepping        : 0

The hard part of setting this, is working out the values of the registers.  It 
takes time to convert the text to binary and get the orderings correct.

Instead, you can generate the XEN config right here:

<div>
    <div>
        <p>
            Simply enter your desired CPU vendor and model string and the XEN config will be generated
        </p>
    </div>

    <div>
        <p>
            <span>CPU Vendor: </span> <input type="text" id="vendor" name="vendor" value="Intrbiz" maxlength="12"/>
            <span> Model: </span>     <input type="text" id="model" name="model" value="Intrbiz XEN virtual CPU" maxlength="48" style="width: 300px;"/>
        </p>
        <pre id="xen-cpuid-config"></pre>
    </div>

    <script type="text/javascript">
        /* <![CDATA[ */
        $(document).ready(function() {
            $('#xen-cpuid-config').html( cpuid($('#vendor').val(), $('#model').val()) );
            $('#vendor').change(function(ev) { $('#xen-cpuid-config').html( cpuid($(this).val(), $('#model').val()) ); });
            $('#model').change(function(ev)  { $('#xen-cpuid-config').html( cpuid($('#vendor').val(), $(this).val()) ); });
            $('#vendor').keydown(function(ev)  { $('#xen-cpuid-config').html( cpuid($(this).val(), $('#model').val()) ); });
            $('#model').keydown(function(ev)   { $('#xen-cpuid-config').html( cpuid($('#vendor').val(), $(this).val()) ); });
            $('#vendor').keyup(function(ev)  { $('#xen-cpuid-config').html( cpuid($(this).val(), $('#model').val()) ); });
            $('#model').keyup(function(ev)   { $('#xen-cpuid-config').html( cpuid($('#vendor').val(), $(this).val()) ); });
        });

        function cpuid(vendor, model)
        {
            vendor = rpad(truncate(vendor, 12), 12);
            model = rpad(truncate(model, 48), 48);
            //
            var sb = ["cpuid = [ "];
            // Vendor
            sb.push("'0:eax=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx");
            sb.push(",ebx=");
            sb.push(textToRegister(vendor.substr(0, 4)));
            sb.push(",ecx=");
            sb.push(textToRegister(vendor.substr(8, 4)));
            sb.push(",edx=");
            sb.push(textToRegister(vendor.substr(4, 4)));
            sb.push("',\n");
            // Features
            sb.push("          '1:eax=00000000000000000000000000000000,ebx=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx,ecx=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx,edx=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx',\n");
            // Model
            sb.push("'36507222018:");
            sb.push("eax=");
            sb.push(textToRegister(model.substr( 0, 4)));
            sb.push(",ebx=");
            sb.push(textToRegister(model.substr( 4, 4)));
            sb.push(",ecx=");
            sb.push(textToRegister(model.substr( 8, 4)));
            sb.push(",edx=");
            sb.push(textToRegister(model.substr(12, 4)));
            sb.push("',\n");
            sb.push("'36507222019:");
            sb.push("eax=");
            sb.push(textToRegister(model.substr(16, 4)));
            sb.push(",ebx=");
            sb.push(textToRegister(model.substr(20, 4)));
            sb.push(",ecx=");
            sb.push(textToRegister(model.substr(24, 4)));
            sb.push(",edx=");
            sb.push(textToRegister(model.substr(28, 4)));
            sb.push("',\n");
            sb.push("'36507222020:");
            sb.push("eax=");
            sb.push(textToRegister(model.substr(32, 4)));
            sb.push(",ebx=");
            sb.push(textToRegister(model.substr(36, 4)));
            sb.push(",ecx=");
            sb.push(textToRegister(model.substr(40, 4)));
            sb.push(",edx=");
            sb.push(textToRegister(model.substr(44, 4)));
            sb.push("'");
            //
            sb.push("]\n");
            return sb.join('');
        }
        
        function textToRegister(text)
        {
            var reg = [];
            for (var i = 3; i >= 0; i--)
            {
                reg.push( charAsBinary(text.charCodeAt(i)) );
            }
            return reg.join('');
        }
        
        function charAsBinary(code)
        {
            var str = [];
            for (var i = 0; i < 8; i++)
            {
                str.push( (code & 128) == 128 ? "1" : "0" );
                code = code << 1;
            }
            return str.join('');
        }
        
        function truncate(text, length)
        {
            return text.length > length ? text.substr(0, length) : text;
        }
        
        function rpad(text, length)
        {
            while (text.length < length)
            {
                text += "\0";
            }
            return text;
        }
        /* ]]> */
    </script>
</div>
