# programacion_asincrona

El link a este repositorio e: [github](https://github.com/GonzaloGmv/programacion_asincrona)

En esta tarea he realizado paso a paso este código que descarga imágenes de forma estándar y de forma asíncrona, para así ver la diferencia entre trabajar en asíncrono o no.

El código final es el siguiente:

### Estándar
```
from bs4 import BeautifulSoup
from urllib.parse import urlparse


def get_images_src_from_html(html_doc):    
    """Recupera todo el contenido de los atributos src de las etiquetas img"""   
    soup = BeautifulSoup(html_doc, "html.parser")    
    return (img.get('src') for img in soup.find_all('img'))

def get_uri_from_images_src(base_uri, images_src):    
    """Devuelve una a una cada URI de la imagen a descargar"""    
    parsed_base = urlparse(base_uri)    
    for src in images_src:    
        parsed = urlparse(src)    
        if parsed.netloc == '':    
            path = parsed.path    
            if parsed.query:    
                path += '?' + parsed.query    
            if path[0] != '/':    
                if parsed_base.path == '/':    
                    path = '/' + path    
                else:    
                    path = '/' + '/'.join(parsed_base.path.split('/')[:-1]) + '/' + path    
            yield parsed_base.scheme + '://' + parsed_base.netloc + path  
        else:    
            yield parsed.geturl() 
```

### Asíncrono
```
from bs4 import BeautifulSoup
import asyncio
from urllib.parse import urlparse
import aiohttp
from os import sep
import sys


async def wget(session, uri):  
    async with session.get(uri) as response:  
        if response.status != 200:  
            return None  
        if response.content_type.startswith("text/"):  
            return await response.text()  
        else:  
            return await response.read()
        
async def download(session, uri):  
    content = await wget(session, uri)  
    if content is None:  
        return None  
    with open(uri.split(sep)[-1], "wb") as f:  
        f.write(content)  
        return uri 

async def get_images_src_from_html(html_doc):  
    soup = BeautifulSoup(html_doc, "html.parser")  
    for img in soup.find_all('img'):  
        yield img.get('src')  
        await asyncio.sleep(0.001)

async def get_uri_from_images_src(base_uri, images_src):  
    """Devuelve una a una cada URI de imagen a descargar"""  
    parsed_base = urlparse(base_uri)  
    async for src in images_src:  
        parsed = urlparse(src)  
        if parsed.netloc == '':  
            path = parsed.path  
            if parsed.query:  
                path += '?' + parsed.query  
            if path[0] != '/':  
                if parsed_base.path == '/':  
                    path = '/' + path  
                else:  
                    path = '/' + '/'.join(parsed_base.path.split('/')[:-1]) + '/' + path  
            yield parsed_base.scheme + '://' + parsed_base.netloc + path  
        else:  
            yield parsed.geturl()  
        await asyncio.sleep(0.001) 

async def get_images(session, page_uri):  
    html = await wget(session, page_uri)  
    if not html:  
        print("Error: no se ha encontrado ninguna imagen", sys.stderr)  
        return None  
    images_src_gen = get_images_src_from_html(html)  
    images_uri_gen = get_uri_from_images_src(page_uri, images_src_gen)  
    async for image_uri in images_uri_gen:  
        print('Descarga de %s' % image_uri)  
        await download(session, image_uri)

async def main():  
    web_page_uri = 'http://www.formation-python.com/'  
    async with aiohttp.ClientSession() as session:  
        await get_images(session, web_page_uri)
    print('Todo ok')

asyncio.run(main()) 
```
