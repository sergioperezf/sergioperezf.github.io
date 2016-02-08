---
layout: post
title:  "Dormir la pantalla programáticamente"
date:   2014-08-16 21:22:00
categories: .net
tags: .net windows
image: /assets/article_images/2014-08-16-dormir-pantalla-.net/lion.jpg
comments: true
disqus_id: 7cc11e27-a869-49ff-8e2b-c8c85c620444
---
<p>Hace varios años, no recuerdo bien dónde, encontré el código para enviar mensajes a la pantalla desde un programa .NET. Con un poco de lectura de la <a href="http://msdn.microsoft.com/en-us/library/aa984739%28v=vs.71%29.aspx">documentación</a> (y <a href="http://msdn.microsoft.com/en-us/library/windows/desktop/ms646360%28v=vs.85%29.aspx">acá</a>), escribí un programa en C# para hacer sólo una cosa: apagar la pantalla de mi computador portátil.</p>
<p>Soy consciente de que esta probalemente no es la manera más elegante de lograr eso, pero desde que lo desarrollé (cosa que no recuerdo me haya tardado más de 5 minutos), puse un acceso directo en mi barra de tareas y es de las cosas más útiles que tengo allí. Cada vez que deseo dejar el portátil prendido pero no deseo esperar a que se apague la pantalla, pulso en el ícono o la combinación de teclas y la pantalla se apaga. A continuación pongo a disposición el código a manera de curiosidad.</p>

~~~ csharp
using System.Runtime.InteropServices;

namespace TurnOffDisplay
{
    class Program
    {
        static public int WM_SYSCOMMAND = 0x0112;
        static public int SC_MONITORPOWER = 0xF170;
        [DllImport("user32.dll")]
        private static extern int SendMessage(int hWnd, int hMsg, int wParam, int lParam);

        static void Main(string[] args)
        {
            SendMessage(-1, WM_SYSCOMMAND, SC_MONITORPOWER, 2);

        }
    }
}
~~~
<p>Para reproducir, crear un proyecto en Visual Studio (puede ser Express) con un único archivo y pegar el código.</p>
