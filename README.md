![image](https://github.com/OmerFarukKurtulus/C-Net-Cafe-Automation/assets/134921186/7c81c90f-f79b-48d5-97ee-d396ef888db1)

using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.Data;
using System.Drawing;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Windows.Forms;
using System.Data.SqlClient;
using MySql.Data;
using MySql.Data.MySqlClient;
using static System.Windows.Forms.DataFormats;

namespace Proje
{
    public partial class Form1 : Form
    {
        static string connectionString = "Server=localhost;Database=cafe;User ID=root;Password=12345;";
        static MySqlConnection mysqlbaglan = new MySqlConnection(connectionString);

        private Timer timer;

        private void timer_Tick(object sender, EventArgs e)
        {
            // timer yeni iş parcığı açıldı
            System.Threading.Thread thread = new System.Threading.Thread(new System.Threading.ParameterizedThreadStart(timer_Tick_parcacik));
            thread.IsBackground = true;
            thread.Start();
        }

        private void sihirliMesaj(string text)
        {
            // yeni iş parcığı açıldı
            System.Threading.Thread thread = new System.Threading.Thread(new System.Threading.ParameterizedThreadStart((object state) =>
            {
                MessageBox.Show(text);
            }));
            thread.IsBackground = true; // arka planda calissin
            thread.Start(); // başlat
        }

        private void timer_Tick_parcacik(object state)
        {
            if (mysqlbaglan.State == System.Data.ConnectionState.Closed)
                mysqlbaglan.Open();

            Boolean sonDakika = false;
            int sonDakikaMasa = -1;
            string sorgu = "SELECT * FROM masalar";
            using (MySqlCommand komut = new MySqlCommand(sorgu, mysqlbaglan))
            {
                using (MySqlDataReader reader = komut.ExecuteReader())
                {
                    while (reader.Read())
                    {
                        int masa = (int)reader["id"];
                        int masadurum = (int)reader["durum"];
                        int sure = (int)reader["sure"];
                        if (masadurum == 1)
                        {
                            if (sure < 99999)
                            {
                                // süreli masa kısmı
                                if (sure - 1 == 0)
                                {
                                    // son dakika olduğunda kapatma için hazırlanma süreci
                                    sonDakika = true;
                                    sonDakikaMasa = masa;
                                }
                                else
                                {
                                    // son dakika değilse sadece süreyi azaltma
                                    // Yeni bir bağlantı nesnesi oluşturun
                                    using (MySqlConnection updateBaglanti = new MySqlConnection(connectionString))
                                    {
                                        updateBaglanti.Open();

                                        string updateSorgu = "UPDATE masalar SET sure = sure - 1 WHERE id = @masa";
                                        using (MySqlCommand updateKomut = new MySqlCommand(updateSorgu, updateBaglanti))
                                        {
                                            updateKomut.Parameters.AddWithValue("@masa", masa);

                                            // Sorguyu çalıştır
                                            updateKomut.ExecuteNonQuery();
                                            //VeritabanindanMasalariYukleAsync();
                                        }
                                    }
                                }
                            }
                            else
                            {
                                // süresiz masa kısmı 99999 küçükse buraya giriyor
                                using (MySqlConnection updateBaglanti = new MySqlConnection(connectionString))
                                {
                                    updateBaglanti.Open();

                                    string updateSorgu = "UPDATE masalar SET tutar = tutar + 2 WHERE id = @masa";
                                    using (MySqlCommand updateKomut = new MySqlCommand(updateSorgu, updateBaglanti))
                                    {
                                        updateKomut.Parameters.AddWithValue("@masa", masa);

                                        // Sorguyu çalıştır
                                        updateKomut.ExecuteNonQuery();
                                        //VeritabanindanMasalariYukleAsync();
                                    }
                                }
                            }
                        }
                    }
                }
            }

            if (sonDakika)
            {
                MasaSureSifirlaAsync($"MASA {sonDakikaMasa}", sonDakikaMasa);
                MasaDurumuGuncelleAsync($"MASA {sonDakikaMasa}", 0, sonDakikaMasa);
                KasaYukleAsync();
            }
            VeritabanindanMasalariYukleAsync();
            // sihirliMesaj("Her 1 saniyede bir çalışan işlem: " + DateTime.Now);
        }
        private void VeritabanindanMasalariYukleAsync()
        {
            if (InvokeRequired)
            {
                Invoke(new MethodInvoker(VeritabanindanMasalariYukle));
            }
            else
            {
                VeritabanindanMasalariYukle();
            }
        }
        private void MasaSureSifirlaAsync(string masaAdi, int masa)
        {
            if (InvokeRequired)
            {
                Invoke(new Action(() => MasaSureSifirla(masaAdi, masa)));
            }
            else
            {
                MasaSureSifirla(masaAdi, masa);
            }
        }
        private void MasaDurumuGuncelleAsync(string masaAdi, int yeniDurum, int masa)
        {
            if (InvokeRequired)
            {
                Invoke(new Action(() => MasaDurumuGuncelle(masaAdi, yeniDurum, masa)));
            }
            else
            {
                MasaDurumuGuncelle(masaAdi, yeniDurum, masa);
            }
        }

        private void KasaYukleAsync()
        {
            if (InvokeRequired)
            {
                Invoke(new MethodInvoker(KasaYukle));
            }
            else
            {
                KasaYukle();
            }
        }
        public Form1()
        {
            InitializeComponent();
            timer = new Timer();
            timer.Interval = 60000;
            timer.Tick += timer_Tick;
            timer.Start();
        }


        private void Form1_Load(object sender, EventArgs e)
        {
            VeritabanindanMasalariYukle();
            VeritabanindanIstekleriYukle();
            KasaYukle();
        }


        private void VeritabanindanMasalariYukle()
        {
            try
            {
                // Bağlantıyı aç (eğer kapalıysa)
                if (mysqlbaglan.State == System.Data.ConnectionState.Closed)
                    mysqlbaglan.Open();

                string sorgu = "SELECT * FROM masalar";

                using (MySqlCommand komut = new MySqlCommand(sorgu, mysqlbaglan))
                {
                    // Temizleme işlemleri
                    // comboBox1.Items.Clear();
                    dataGridView1.Rows.Clear();
                    int ilkYukleme = 0;
                    if (comboBox1.Items.Count < 1)
                    {
                        ilkYukleme = 1;
                    }
                    using (MySqlDataReader reader = komut.ExecuteReader())
                    {
                        while (reader.Read())
                        {
                            // ComboBox'a masa ekleniyor
                            if (ilkYukleme == 1)
                            {
                                comboBox1.Items.Add(new MasaItem { MasaAdi = reader["masa"].ToString(), MasaTag = (int)reader["id"] });
                            }
                            if ((int)reader["durum"] == 0)
                            {
                                ButonlaraRenkVer((int)reader["id"], Color.DarkRed);
                            }
                            else
                            {
                                ButonlaraRenkVer((int)reader["id"], Color.Green);
                            }

                            // DataGridView'a veri ekleniyor
                            string durum = (int)reader["durum"] == 1 ? "Açık" : "Kapalı";
                            string surestring = (int)reader["sure"] < 99999 ? reader["sure"].ToString() : "Süresiz";
                            object[] rowData = { reader["id"], reader["masa"], durum, surestring, reader["tutar"] };
                            dataGridView1.Rows.Add(rowData);
                        }
                    }
                }
            }
            catch (Exception ex)
            {
                sihirliMesaj("Hata: " + ex.Message);
            }
        }

        private void VeritabanindanIstekleriYukle()
        {
            try
            {
                // Bağlantıyı aç (eğer kapalıysa)
                if (mysqlbaglan.State == System.Data.ConnectionState.Closed)
                    mysqlbaglan.Open();

                string sorgu = "SELECT * FROM istekler";

                using (MySqlCommand komut = new MySqlCommand(sorgu, mysqlbaglan))
                {
                    // Temizleme işlemleri
                    dataGridView2.Rows.Clear();

                    using (MySqlDataReader reader = komut.ExecuteReader())
                    {
                        while (reader.Read())
                        {
                            // DataGridView'a veri ekleniyor
                            object[] rowData = { reader["id"], reader["masa"], reader["istek"], reader["ucret"], reader["tarih"] };
                            dataGridView2.Rows.Add(rowData);
                        }
                    }
                }
            }
            catch (Exception ex)
            {
               sihirliMesaj("Hata: " + ex.Message);
            }
        }
        private void KasaYukle()
        {
            try
            {
                // Bağlantıyı aç (eğer kapalıysa)
                if (mysqlbaglan.State == System.Data.ConnectionState.Closed)
                    mysqlbaglan.Open();

                string sorgu = "SELECT kasa FROM kasa WHERE id = 0";

                using (MySqlCommand komut = new MySqlCommand(sorgu, mysqlbaglan))
                {
                    int kasa = (int)komut.ExecuteScalar();
                    button24.Text = $"{kasa} ₺";
                }
            }
            catch (Exception ex)
            {
               sihirliMesaj("Hata: " + ex.Message);
            }
        }

        private bool MasaDurumuGuncelle(string masaAdi, int yeniDurum, int masaTag)
        {
            try
            {
                // Bağlantıyı aç (eğer kapalıysa)
                if (mysqlbaglan.State == System.Data.ConnectionState.Closed)
                    mysqlbaglan.Open();

                string durumSorgu = "SELECT durum FROM masalar WHERE id = @masaTag";

                using (MySqlCommand durumKomut = new MySqlCommand(durumSorgu, mysqlbaglan))
                {
                    durumKomut.Parameters.AddWithValue("@masaTag", masaTag);

                    // Durumu oku
                    int durum = (int)durumKomut.ExecuteScalar();

                    // Durum 0 ise eklemeyi yapma
                    if (durum == 1)
                    {
                        if (yeniDurum == 1)
                        {
                           sihirliMesaj($"Masayı açabilmek için öncelikle kapalı olması gerekir.");
                            return false;
                        }
                    }
                    if (durum == 0)
                    {
                        if (yeniDurum == 0)
                        {
                           sihirliMesaj($"Masayı kapatabilmek için öncelikle açık olması gerekir.");
                            return false;
                        }
                    }
                }
                // Belirli masa adına sahip kaydın durumunu güncelle
                string updateSorgu = "UPDATE masalar SET durum = @yeniDurum WHERE masa = @masaAdi";

                using (MySqlCommand updateKomut = new MySqlCommand(updateSorgu, mysqlbaglan))
                {
                    updateKomut.Parameters.AddWithValue("@yeniDurum", yeniDurum);
                    updateKomut.Parameters.AddWithValue("@masaAdi", masaAdi);

                    // Sorguyu çalıştır
                    updateKomut.ExecuteNonQuery();
                    if (yeniDurum == 1)
                    {
                        ButonlaraRenkVer(masaTag, Color.Green);
                    }
                    else
                    {
                        ButonlaraRenkVer(masaTag, Color.DarkRed);
                    }
                    return true;
                }
            }
            catch (Exception ex)
            {
               sihirliMesaj("Hata: " + ex.Message);
                return false;
            }
        }

        private void MasaSureEkle(int type, string masaAdi, int eklenenSure, int masaTag)
        {
            try
            {
                // Bağlantıyı aç (eğer kapalıysa)
                if (mysqlbaglan.State == System.Data.ConnectionState.Closed)
                    mysqlbaglan.Open();

                // Belirli masa adına sahip kaydın durumunu kontrol et
                string durumSorgu = "SELECT durum FROM masalar WHERE id = @masaTag";

                using (MySqlCommand durumKomut = new MySqlCommand(durumSorgu, mysqlbaglan))
                {
                    durumKomut.Parameters.AddWithValue("@masaTag", masaTag);

                    // Durumu oku
                    int durum = (int)durumKomut.ExecuteScalar();

                    // Durum 0 ise eklemeyi yapma
                    if (durum == 0)
                    {
                       sihirliMesaj($"Masanın durumu kapalı olduğu için süre eklenmedi.");
                        return;
                    }
                }

                string sureSorgu = "SELECT sure FROM masalar WHERE id = @masaTag";

                using (MySqlCommand sureKomut = new MySqlCommand(sureSorgu, mysqlbaglan))
                {
                    sureKomut.Parameters.AddWithValue("@masaTag", masaTag);

                    int sure = (int)sureKomut.ExecuteScalar();
                    if (sure > 99998)
                    {
                       sihirliMesaj($"Süresiz masaya süre eklenemez.");
                        return;
                    }
                    if (sure != 0 && sure < 99999 && eklenenSure > 99998)
                    {
                        sihirliMesaj($"Süreli masaya süresiz oturum eklenemez.");
                        return;
                    }
                }
                int birimFiyat = 2;
                if (eklenenSure > 99998)
                {
                    birimFiyat = 0;
                }
                int ucret = eklenenSure * birimFiyat;
                string updateSorgu = "UPDATE masalar SET sure = sure + @eklenenSure, tutar = tutar + @ucret WHERE id = @masaTag AND durum = 1";

                using (MySqlCommand updateKomut = new MySqlCommand(updateSorgu, mysqlbaglan))
                {
                    updateKomut.Parameters.AddWithValue("@ucret", ucret);
                    updateKomut.Parameters.AddWithValue("@eklenenSure", eklenenSure);
                    updateKomut.Parameters.AddWithValue("@masaTag", masaTag);

                    // Sorguyu çalıştır
                    updateKomut.ExecuteNonQuery();

                    if (type == 0)
                    {
                        string surestring = eklenenSure < 99999 ? eklenenSure.ToString() : "Süresiz";
                        sihirliMesaj($"Masalar Açıldı\nMASA:{comboBox1.Text} SÜRE:{surestring}DK");
                    }
                    else if (type == 1)
                    {
                        sihirliMesaj($"Süre Eklendi\nMasa:{comboBox1.Text} Eklenen Süre:{eklenenSure.ToString()}DK");
                    }
                }
            }
            catch (Exception ex)
            {
               sihirliMesaj("Hata: " + ex.Message);
            }
        }

        private void MasaSureSifirla(string masaAdi, int masa)
        {
            try
            {
                // Bağlantıyı aç (eğer kapalıysa)
                if (mysqlbaglan.State == System.Data.ConnectionState.Closed)
                    mysqlbaglan.Open();

                int alinicak_odeme = 0;

                string tutarSorgu = "SELECT tutar FROM masalar WHERE id = @masa";

                using (MySqlCommand tutarKomut = new MySqlCommand(tutarSorgu, mysqlbaglan))
                {
                    tutarKomut.Parameters.AddWithValue("@masa", masa);

                    // Durumu oku
                    int tutar = (int)tutarKomut.ExecuteScalar();
                    alinicak_odeme = tutar;
                    string kasaSorgu = "UPDATE kasa SET kasa = kasa + @tutar";

                    using (MySqlCommand kasaKomut = new MySqlCommand(kasaSorgu, mysqlbaglan))
                    {
                        kasaKomut.Parameters.AddWithValue("@tutar", tutar);

                        // Sorguyu çalıştır
                        kasaKomut.ExecuteNonQuery();
                    }
                }

                string kasaSorgu2 = "SELECT kasa FROM kasa WHERE id = 0";

                using (MySqlCommand kasaKomut2 = new MySqlCommand(kasaSorgu2, mysqlbaglan))
                {
                    kasaKomut2.Parameters.AddWithValue("@masa", masa);

                    // Durumu oku
                    int kasa = (int)kasaKomut2.ExecuteScalar();
                    button24.Text = $"{kasa} ₺";
                }

                string deleteSorgu = "DELETE FROM istekler WHERE masa = @masa";
                using (MySqlCommand deleteKomut = new MySqlCommand(deleteSorgu, mysqlbaglan))
                {
                    deleteKomut.Parameters.AddWithValue("@masa", masa);
                    // Sorguyu çalıştır
                    deleteKomut.ExecuteNonQuery();
                    VeritabanindanIstekleriYukle();
                }

                string updateSorgu = "UPDATE masalar SET sure = 0, tutar = 0 WHERE id = @masa";

                using (MySqlCommand updateKomut = new MySqlCommand(updateSorgu, mysqlbaglan))
                {
                    updateKomut.Parameters.AddWithValue("@masa", masa);

                    // Sorguyu çalıştır
                    updateKomut.ExecuteNonQuery();
                }
                sihirliMesaj($"Masa kapatıldı\nMasa:{masaAdi}\nHesap:{alinicak_odeme.ToString()}₺");
            }
            catch (Exception ex)
            {
               sihirliMesaj("Hata: " + ex.Message);
            }
        }

        private void MasaAc()
        {
            // Panel içindeki tüm RadioButton kontrolerini döngü ile gez
            foreach (RadioButton radioButton in panel1.Controls.OfType<RadioButton>())
            {
                if (radioButton.Checked)
                {
                    MasaItem selectedMasa = comboBox1.SelectedItem as MasaItem;
                    if (selectedMasa != null)
                    {
                        int masaTag = selectedMasa.MasaTag;
                        // Seçili olan RadioButton'u getir
                        int convertedNumber;
                        if (radioButton.Text == "Süresiz")
                        {
                            bool masa_durum = MasaDurumuGuncelle(comboBox1.Text, 1, masaTag);
                            if (masa_durum == false) return;
                            MasaSureEkle(0, comboBox1.Text, 99999, masaTag);
                            VeritabanindanMasalariYukle();
                            comboBox1.SelectedIndex = -1;
                            comboBox1.Text = "";
                        }
                        else
                        {
                            int.TryParse(radioButton.Text, out convertedNumber);
                            bool masa_durum = MasaDurumuGuncelle(comboBox1.Text, 1, masaTag);
                            if (masa_durum == false) return;
                            MasaSureEkle(0, comboBox1.Text, convertedNumber, masaTag);
                            VeritabanindanMasalariYukle();
                            comboBox1.SelectedIndex = -1;
                            comboBox1.Text = "";
                        }
                    }
                    break;
                }
            }
        }

        private void SureEkle()
        {
            // Panel içindeki tüm RadioButton kontrolerini döngü ile gez
            foreach (RadioButton radioButton in panel1.Controls.OfType<RadioButton>())
            {
                if (radioButton.Checked)
                {
                    MasaItem selectedMasa = comboBox1.SelectedItem as MasaItem;
                    if (selectedMasa != null)
                    {
                        if (radioButton.Text == "Süresiz")
                        {
                            int masaTag = selectedMasa.MasaTag;
                            // Seçili olan RadioButton'u getir
                            MasaSureEkle(1, comboBox1.Text, 99999, masaTag);
                            VeritabanindanMasalariYukle();
                            comboBox1.SelectedIndex = -1;
                            comboBox1.Text = "";
                        }
                        else
                        {
                            int masaTag = selectedMasa.MasaTag;
                            // Seçili olan RadioButton'u getir
                            int convertedNumber;
                            int.TryParse(radioButton.Text, out convertedNumber);
                            MasaSureEkle(1, comboBox1.Text, convertedNumber, masaTag);
                            VeritabanindanMasalariYukle();
                            comboBox1.SelectedIndex = -1;
                            comboBox1.Text = "";
                        }

                    }
                    break;
                }
            }
        }

        private void MasaKapat()
        {
            // Panel içindeki tüm RadioButton kontrolerini döngü ile gez
            MasaItem selectedMasa = comboBox1.SelectedItem as MasaItem;
            if (selectedMasa != null)
            {
                int masaTag = selectedMasa.MasaTag;
                // Seçili olan RadioButton'u getir
                bool masa_durum = MasaDurumuGuncelle(comboBox1.Text, 0, masaTag);
                if (masa_durum == false) return;

                MasaSureSifirla(comboBox1.Text, masaTag);
                VeritabanindanMasalariYukle();
                comboBox1.SelectedIndex = -1;
                comboBox1.Text = "";
            }
        }

        private void bMasaAc_click(object sender, EventArgs e)
        {
            MasaAc();
        }

        private void SureEkle_click(object sender, EventArgs e)
        {
            SureEkle();
        }

        private void MasaKapat_click(object sender, EventArgs e)
        {
            MasaKapat();
        }

        private void ButonlaraRenkVer(int tag, Color renk)
        {
            foreach (Control control in this.Controls)
            {
                if (control is Button && ((Button)control).Tag != null && ((Button)control).Tag.ToString() == $"masa_{tag.ToString()}")
                {
                    ((Button)control).BackColor = renk;
                }
            }
        }

        public class MasaItem
        {
            public string MasaAdi { get; set; }
            public int MasaTag { get; set; }

            public override string ToString()
            {
                return MasaAdi;
            }
        }

        private void label1_Click(object sender, EventArgs e)
        {

        }

        private void button12_Click(object sender, EventArgs e)
        {

        }

        private void button18_Click(object sender, EventArgs e)
        {

        }

        private void contextMenuStrip1_Opening(object sender, CancelEventArgs e)
        {

        }
        Button btn;

        private void SecilileneGore(object sender, MouseEventArgs e)
        {
            btn = sender as Button;

        }

        private void listView1_SelectedIndexChanged(object sender, EventArgs e)
        {

        }

        private void button1_Click(object sender, EventArgs e)
        {

        }

        private void button23_Click(object sender, EventArgs e)
        {
            Form2 form2 = new Form2();
            form2.Show();
            this.Hide();
        }

        private void dataGridView1_CellContentClick(object sender, DataGridViewCellEventArgs e)
        {

        }

        private void textBox1_TextChanged(object sender, EventArgs e)
        {

        }

        private void label1_Click_1(object sender, EventArgs e)
        {

        }

        private void panel1_Paint(object sender, PaintEventArgs e)
        {

        }

        private void comboBox1_SelectedIndexChanged(object sender, EventArgs e)
        {

        }

        private void rbsuresiz_CheckedChanged(object sender, EventArgs e)
        {

        }
    }
}

